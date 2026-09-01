---
id: SOCIAL-29
title: In-app photo picker - PhotoPickerComponent and AlbumPickerComponent embedded as your own page
industry: 14_social_communication
doc: huawei_industry_tree/14_social_communication/docs/29_custom_album_style.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/custom_album_style-0000002333520485
sample: huawei_industry_tree/14_social_communication/downloads/CustomAlbumStyle.zip
kits: ["@kit.AbilityKit", "@kit.ArkData", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.MediaLibraryKit", "@kit.PerformanceAnalysisKit"]
apis: [PhotoPickerComponent, AlbumPickerComponent, PickerOptions, AlbumPickerOptions, PickerController, DataType, ItemInfo, ClickType, ItemType, dataSharePredicates, "photoAccessHelper.getPhotoAccessHelper", getAssets, getThumbnail, NavPathStack, pushPathByName, PopInfo, Tabs, ForEach]
permissions: []
min_api: 20
modules: [entry (entry)]
findings: [HW-14-0064, HW-14-0065, HW-14-0018, HW-14-0001, HW-14-0087]
status: verified-with-fixes
---

## When to use

Load this card when the app needs a **photo picker that looks like part of the
app** - a chat composer, a post editor, an avatar chooser - rather than the
system picker sheet. The pattern: host `PhotoPickerComponent` and
`AlbumPickerComponent` inside your own `NavDestination`, style them through
`PickerOptions`, and keep the selection in your own state so the page can draw
a tray, a counter and a confirm button around them.

The trade is explicit. The system `PhotoViewPicker` needs no permission and
gives no control; these two components also need no permission (the picker
process owns the media access) but let you place tabs, a preselection, a
maximum count and your own brand colour on top. You get the chrome; you also
become responsible for the selection bookkeeping, and that is where this sample
goes wrong.

It generalises to any "pick media, then come back with it" flow. The important
transferable pieces are the `PickerController` message channel - the only way
to push state *into* the picker after it is mounted - and the discipline of
never destroying what the user already has until the picker returns a result.

## Feature checklist

- A chat page with two seeded bubbles and a `+` button in the input row.
- Tapping `+` opens a full-page picker inside the app's own navigation stack.
- Four tabs: 全部 (all), 视频 (video), 照片 (photos), 分类 (albums).
- The first three tabs are the same picker component with a different
  `MIMEType`; the fourth is the album list.
- Picking an album switches every picker tab to that album and renames the
  header.
- Selected items appear in a horizontal tray at the bottom, numbered, each
  removable by its badge.
- The tray, the count badge and the picker's own checkboxes stay in step in
  both directions.
- Confirming pops back to the chat with the chosen URIs and renders them as
  sent images.
- Cancelling must leave the chat exactly as it was.

## Architecture

One `entry` module. Two pages, one reusable picker wrapper, one model class.

```
entry/src/main/ets
├── common/Constants.ets                 layout numbers, tab labels, MAX_SELECT_NUMBER = 99
├── components/PhotoPickerView .ets      the picker wrapper: options, callbacks, selection bookkeeping
├── entryability/EntryAbility.ets        full-screen layout, avoid areas -> AppStorage
├── entrybackupability/EntryBackupAbility.ets
├── model/SelectPhotoInfo.ets            uri + media type + thumbnail PixelMap
├── pages/ChatPage.ets                   @Entry, the chat and the '+' button
├── pages/SelectPhotoPage.ets            NavDestination: four tabs, the tray, the confirm button
└── utils/AppRouter.ets                  singleton NavPathStack wrapper (push/pop/popWithParam)
```

The documented tree does **not** match the zip: the doc lists
`components/PhotoPickerView.ets`, the archive ships
`components/PhotoPickerView .ets` - with a space before the extension, and the
import in `SelectPhotoPage.ets` carries the same space
(`from '../components/PhotoPickerView '`). It compiles, and it is one rename
away from not compiling. This is one of the four social trees that disagree
with their zips (`HW-14-0001`).

Routing is a route map, not `pages`: `main_pages.json` lists only
`pages/ChatPage`, and `route_map.json` registers `SelectPhotoPage` against
`selectPhotoPageBuilder`. That is why the picker page is a plain `@Component`
with an exported `@Builder` function rather than an `@Entry`.

**The design decision worth copying** is `@Provide` / `@Consume` for the
selection. `SelectPhotoPage` owns `@Provide selectSelectPhotoInfos`, and every
one of the four `PhotoPickerView` instances takes it as
`@Consume @Watch('refreshSelectPhoto')`. There are three live picker components
on the page at once (all / video / photo), each with its own internal checkbox
state, and one shared array. When any of them - or the tray's remove badge -
mutates the array, `@Watch` fires in all of them and each pushes the new URI
list back into its own picker through `pickerController.setData(...)`. One
source of truth, four views, no cross-component wiring. Copy that shape.

## Implementation steps

1. **Register the picker page in `route_map.json`** and export a `@Builder`
   function for it; push with `pushPathByName(name, param, onPop)` so the page
   can return a result.
2. **Do not clear the destination state before pushing.** Clear it in the
   `onPop` callback, from the returned result (`HW-14-0064`) - back-press pops
   without a result, and the callback then never runs.
3. **Configure `PickerOptions` in `aboutToAppear`**, before the component
   mounts: `MIMEType`, `maxSelectNumber`, `isSearchSupported`,
   `isPhotoTakingSupported`, `checkBoxColor`, and `preselectedUris` from the
   current selection.
4. **Hold a `PickerController`** and wait for `onPickerControllerReady` before
   calling `setData` - earlier calls are dropped.
5. **Push album changes in, don't remount:** `AlbumPickerComponent`'s
   `onAlbumClick` writes `albumUri`, and a `@Watch` on it sends
   `DataType.SET_ALBUM_URI` to every picker instance.
6. **Return `true` from `onItemClicked` only when the item is really
   selected** - the boolean is the checkbox's answer. Resolve the asset query
   first and handle its failure (`HW-14-0065`).
7. **Close every `FetchResult`.** `getAssets` hands back a native cursor; one
   leaks per pick in the shipped code (`HW-14-0065`).
8. **Key both `ForEach`s by something unique.** `ChatPage` keys `SelectPhotoInfo`
   objects with `(item: string) => item`, which stringifies every item to
   `[object Object]` - one key for the whole list (`HW-14-0018`).
9. **Pop with the result**, `pathStack.pop(param)`, and read it in the pusher's
   `onPop` as `popInfo.result`.

## Verified snippets

All snippets are from `CustomAlbumStyle.zip`. Corrected forms are marked.

**The picker wrapper and its controller channel — `entry/src/main/ets/components/PhotoPickerView .ets`** (as shipped)

```typescript
@Component
export struct PhotoPickerView {
  @Consume @Watch('refreshSelectPhoto') selectSelectPhotoInfos: Array<SelectPhotoInfo>;
  @State pickerController: PickerController = new PickerController();
  @Prop @Watch('onAlbumUriRefresh') albumUri: string;
  @Prop mimeType: photoAccessHelper.PhotoViewMIMETypes;
  pickerOptions: PickerOptions = new PickerOptions();

  aboutToAppear(): void {
    this.pickerOptions.MIMEType = this.mimeType;
    this.pickerOptions.maxSelectNumber = Constants.MAX_SELECT_NUMBER;
    this.pickerOptions.isSearchSupported = false;
    this.pickerOptions.isPhotoTakingSupported = false;
    this.pickerOptions.checkBoxColor = Constants.BUTTON_BACKGROUND_COLOR;
    let selectSelectPhotoInfoList: string[] = [];
    for (let selectSelectPhotoInfosElement of this.selectSelectPhotoInfos) {
      selectSelectPhotoInfoList.push(selectSelectPhotoInfosElement.uri);
    }
    this.pickerOptions.preselectedUris = [...selectSelectPhotoInfoList];
  }

  refreshSelectPhoto() {                       // fires on any change to the shared array
    let selectSelectPhotoInfoList: string[] = [];
    for (let selectSelectPhotoInfosElement of this.selectSelectPhotoInfos) {
      selectSelectPhotoInfoList.push(selectSelectPhotoInfosElement.uri);
    }
    this.pickerController.setData(DataType.SET_SELECTED_URIS, selectSelectPhotoInfoList);
  }

  onAlbumUriRefresh(): void {
    if (this.albumUri !== '') {
      this.pickerController.setData(DataType.SET_ALBUM_URI, this.albumUri);
    }
  }

  private onPickerControllerReady(): void {
    let elements: number[] = [PhotoBrowserUIElement.BACK_BUTTON];
    this.pickerController.setPhotoBrowserUIElementVisibility(elements, false);
  }
}
```

**Two channels, deliberately different.** `preselectedUris` is a *creation*
parameter: it is read once, when the component mounts, so it only covers the
case of reopening the picker with an existing selection. Everything after mount
goes through `pickerController.setData` - `SET_SELECTED_URIS` for the
checkboxes, `SET_ALBUM_URI` for the visible album. The component renders in its
own process; there is no way to reach into it declaratively, which is why the
controller exists and why `onPickerControllerReady` matters - `setData` calls
made before that callback are silently dropped.

`maxSelectNumber` is 99 here, and `isSearchSupported` / `isPhotoTakingSupported`
are both `false`: the sample deliberately strips the picker down to a grid so
that its own tab bar is the only navigation.

**Selecting an item — same file** (corrected, see `HW-14-0065`)

```typescript
async createSelectPhotoInfo(selectPhotoInfo: SelectPhotoInfo) {
  let phAccessHelper = photoAccessHelper.getPhotoAccessHelper(this.context);
  let predicates: dataSharePredicates.DataSharePredicates = new dataSharePredicates.DataSharePredicates();
  predicates.equalTo('uri', selectPhotoInfo.uri);          // query by the uri the picker handed back
  let fetchOptions: photoAccessHelper.FetchOptions = {
    fetchColumns: ['media_type', 'date_added'],
    predicates: predicates
  };
  let fetchResult = await phAccessHelper.getAssets(fetchOptions);
  try {
    let photoAsset: photoAccessHelper.PhotoAsset = await fetchResult.getFirstObject();
    selectPhotoInfo.type = photoAsset.get('media_type') as number;
    selectPhotoInfo.thumbnail = await photoAsset.getThumbnail();
  } finally {
    fetchResult.close();                                   // FIX: the sample never closes the cursor
  }
}

private onItemClicked(itemInfo: ItemInfo, clickType: ClickType): boolean {
  if (!itemInfo) {
    return false;
  }
  let type: ItemType | undefined = itemInfo.itemType;
  let uri: string | undefined = itemInfo.uri;
  if (type === ItemType.CAMERA) {
    return true;                                           // true here launches the system camera
  }
  if (clickType === ClickType.SELECTED) {
    if (uri) {
      let info = this.selectSelectPhotoInfos.find((item) => item.uri === uri);
      if (info === undefined) {
        let selectPhotoInfo: SelectPhotoInfo = new SelectPhotoInfo();
        selectPhotoInfo.uri = uri;
        this.createSelectPhotoInfo(selectPhotoInfo)
          .then(() => {
            this.selectSelectPhotoInfos.push(selectPhotoInfo);
          })
          .catch(() => {                                   // FIX: the sample has no catch
            this.selectSelectPhotoInfos =
              this.selectSelectPhotoInfos.filter((item: SelectPhotoInfo) => item.uri !== uri);
          });
      }
    }
    return true;                                           // true = draw the checkbox
  }
  if (uri) {
    this.selectSelectPhotoInfos = this.selectSelectPhotoInfos.filter((item: SelectPhotoInfo) => item.uri !== uri);
  }
  return true;
}
```

**The return value of `onItemClicked` is the checkbox.** Returning `true` ticks
the item; returning `false` refuses the tick. The shipped code returns `true`
synchronously and starts the asset query in the background, so on any
media-library failure the picker shows a checked photo the result array does
not contain - and the user confirms a selection that is one item short. The
corrected form keeps the optimistic tick (the query is fast and blocking the
gesture would feel worse) but reconciles on failure by removing the entry
again, which `@Watch` immediately reflects back into the picker's checkboxes.

Two other details are load-bearing: `getAssets` returns a native cursor that
must be `close()`d - one leaks per pick in the shipped code - and the
deselection branch replaces the array (`filter`) while the selection branch
mutates it (`push`). Both trigger `@Watch` here because the array is a
`@Provide`, but replacing is the safer habit.

**The '+' button — `entry/src/main/ets/pages/ChatPage.ets`** (corrected, see `HW-14-0064`, `HW-14-0018`)

```typescript
ForEach(this.selectSelectPhotoInfos, (item: SelectPhotoInfo, index: number) => {
  ListItem() {
    this.imageBuilder(item.type === Constants.TYPE_IMAGE ? item.uri : item.thumbnail!, index);
  };
}, (item: SelectPhotoInfo, index: number) => `${item.uri}_${index}`)   // FIX: sample used (item: string) => item

// ...

Image($r('app.media.ic_public_add'))
  .width(Constants.IMAGE_WIDTH)
  .height(Constants.IMAGE_HEIGHT)
  .onClick(() => {
    AppRouter.pushByName(Constants.PUSH_NAME, 0, (popInfo: PopInfo) => {
      // FIX: the sample cleared selectSelectPhotoInfos here, before the push
      this.selectSelectPhotoInfos = popInfo.result as SelectPhotoInfo[];
    });
  });
```

**Clearing before the push is the bug that costs data.** The shipped handler
sets `this.selectSelectPhotoInfos = []` and *then* pushes the picker. The
callback that would refill it only runs when the page pops **with a
parameter** - and `SelectPhotoPage.onBackPressed` calls `AppRouter.pop()` with
no argument. So the sequence "send three photos, tap `+`, press back" ends with
an empty chat. The single assignment in `onPop` is both the clear and the
refill; nothing needs to run before the push.

The key generator is the second defect in four lines. `(item: string) => item`
is applied to `SelectPhotoInfo` objects, so every key stringifies to
`[object Object]` and ArkUI's diffing sees one item no matter how many were
sent. This is a pattern across six chat samples in this industry
(`HW-14-0018`); `uri + index` is the minimal fix, a message id is the right
one. Note that `SelectPhotoPage`'s own tray keys with `JSON.stringify(item)`,
which is unique but rebuilds a whole thumbnail row on any field change.

**The four tabs and the album list — `entry/src/main/ets/pages/SelectPhotoPage.ets`** (as shipped)

```typescript
@Provide selectSelectPhotoInfos: Array<SelectPhotoInfo> = [];
@State albumUri: string = '';
albumPickerOptions: AlbumPickerOptions = new AlbumPickerOptions();

private onAlbumClick(albumInfo: AlbumInfo): boolean {
  if (albumInfo?.uri) {
    this.albumUri = albumInfo.uri;      // @Watch in each PhotoPickerView pushes SET_ALBUM_URI
    this.currentTabIndex = 0;           // and jump back to the 全部 tab
    this.selectedTabIndex = 0;
  }
  if (albumInfo?.albumName) {
    this.albumName = albumInfo.albumName;
  }
  return true;
}

// ... inside Tabs:
TabContent() {
  PhotoPickerView({ mimeType: photoAccessHelper.PhotoViewMIMETypes.IMAGE_VIDEO_TYPE, albumUri: this.albumUri });
}
.tabBar(this.tabBuilder(0, Constants.TAB_ALL));

TabContent() {
  Column() {
    AlbumPickerComponent({
      albumPickerOptions: this.albumPickerOptions,
      onAlbumClick: (albumInfo: AlbumInfo): boolean => this.onAlbumClick(albumInfo)
    })
      .height('70%')
      .width('100%')
      .align(Alignment.Top);
  }
}
.tabBar(this.tabBuilder(3, Constants.TAB_CLASSIFY));
```

**The three media tabs are one component with three MIME types.** All /
video / photo differ only in `PhotoViewMIMETypes`, and because each instance
consumes the same `@Provide` array, a photo ticked on the 全部 tab is already
ticked when the user swipes to 照片. The album tab is the fourth `TabContent`
rather than a dropdown, which is why `onAlbumClick` also resets
`currentTabIndex` to 0 - selecting an album is meant to feel like navigating
back to the grid.

`onAlbumClick` returning `true` tells the album component the click was
handled. `AlbumPickerOptions` is left at its defaults here; it is where a
`themeColorMode` or a filtered album subtype would go.

## Permissions & config

**None.** The sample declares no `requestPermissions`, and it does not need
any: `PhotoPickerComponent` and `AlbumPickerComponent` render in the picker's
own process and hand back URIs the app is temporarily authorised to read. That
is the main reason to prefer them over `photoAccessHelper` queries against the
whole library, which would need `ohos.permission.READ_IMAGEVIDEO`.

The one place the sample does touch the media library directly -
`getPhotoAccessHelper(...).getAssets(...)` in `createSelectPhotoInfo` - works
only because the predicate is `equalTo('uri', ...)` on a URI the picker just
granted.

`deviceTypes` is `phone`, `tablet`, `2in1`. `routerMap` is `$profile:route_map`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- `maxSelectNumber` is 99, and every pick decodes a thumbnail `PixelMap` held
  in the selection array. At the top of that range the tray holds 99 decoded
  bitmaps with no recycling.
- `onPageHide` clears the selection, so backgrounding the app mid-pick loses
  it.
- The chat side is a demo: two hardcoded bubbles, the text field is an empty
  `Text` with a background, and there is no send action beyond appending the
  picked images.
- The album tab's lower 30% is an empty `Row` whose `onClick` toggles an
  `openAlbum` flag that nothing reads.

## Pitfalls

- **`HW-14-0064`** (B/medium, confirmed): tapping `+` clears
  `selectSelectPhotoInfos` before pushing the picker, and
  `SelectPhotoPage.onBackPressed` pops without a parameter so the `onPop`
  callback never runs - every photo already in the chat is gone. Fix: move the
  clear into the `onPop` callback.
- **`HW-14-0065`** (B/low, confirmed): `onItemClicked` returns `true` at once
  while `createSelectPhotoInfo` runs unguarded, so a failed asset query leaves
  a ticked checkbox with no matching entry in the result; the `FetchResult` is
  also never closed, leaking a native cursor per pick. Fix: `then`/`catch`
  around `createSelectPhotoInfo` and `fetchResult.close()`.
- **`HW-14-0018`** (B/medium, confirmed): systematic broken `ForEach` keys
  across six chat samples; here `ChatPage.ets` keys `SelectPhotoInfo` objects
  with `(item: string) => item`, collapsing every item to `[object Object]`.
  Fix: append the index or use a message id.
- **`HW-14-0001`** (E/low, confirmed): systematic project-tree drift; this
  doc lists `components/PhotoPickerView.ets` while the zip ships
  `PhotoPickerView .ets` with a space in the name. Fix: regenerate the 工程目录
  block (and rename the file).

## References

- `documentation/harmonyos-references/04_media/ohos-file-photopickercomponent.md` - `PhotoPickerComponent`, `PickerOptions`, `PickerController`, `ItemInfo`, `ClickType`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ohos-file-photopickercomponent
- `documentation/harmonyos-references/04_media/ohos-file-albumpickercomponent.md` - `AlbumPickerComponent`, `AlbumPickerOptions`, `AlbumInfo`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ohos-file-albumpickercomponent
- `documentation/harmonyos-guides/05_media/medialibrary-pickercontroller.md` - the `setData` message channel and `onPickerControllerReady`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/medialibrary-pickercontroller
- `documentation/harmonyos-guides/05_media/component-guidelines-albumpicker.md` - embedding the album list
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/component-guidelines-albumpicker
- `documentation/harmonyos-references/04_media/js-apis-photoAccessHelper.md` - `getAssets`, `FetchResult`, `PhotoAsset.getThumbnail`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-photoaccesshelper
- `documentation/harmonyos-guides/03_application-framework/arkts-navigation-navigation.md` - route maps, `pushPathByName` and `PopInfo`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-navigation-navigation
- `SOCIAL-08` - the systematic `ForEach` key defect this sample is an instance of
