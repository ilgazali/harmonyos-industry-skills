---
id: OFFICE-16
title: ID-photo recommendation - PhotoViewPicker with recommendationType so the gallery pre-filters ID cards
industry: 05_office
doc: huawei_industry_tree/05_office/docs/16_recommend_id_photos.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/recommend_id_photos-0000002344341185
sample: huawei_industry_tree/05_office/downloads/RecommendPhoto.zip
kits: ["@kit.MediaLibraryKit", "@kit.ArkUI", "@kit.AbilityKit", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit"]
apis: ["photoAccessHelper.PhotoViewPicker", "PhotoViewPicker.select", "photoAccessHelper.PhotoSelectOptions", "photoAccessHelper.RecommendationOptions", "photoAccessHelper.RecommendationType", "photoAccessHelper.PhotoViewMIMETypes", "photoAccessHelper.PhotoSelectResult", Image, ImageContent, ImageFit, Stack, "window.setWindowLayoutFullScreen", "window.getWindowAvoidArea", "window.AvoidAreaType.TYPE_SYSTEM", "window.on('avoidAreaChange')", "window.off('avoidAreaChange')", "AppStorage.setOrCreate", "@StorageLink", "UIContext.px2vp"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-05-0090, HW-05-0091, HW-05-0092, HW-05-0185]
status: verified-with-fixes
---

## When to use

Load this card when a registration or identity-verification flow asks the user
to **upload an ID card photo from the gallery**, and you want the gallery to open
already filtered to the pictures that look like ID cards instead of showing the
whole camera roll.

The entire mechanism is one field on the picker options:

```ts
recommendationOptions: { recommendationType: photoAccessHelper.RecommendationType.ID_CARD }
```

`PhotoViewPicker` is a system picker, so this needs **no permission** - no
gallery read, no camera. The picker returns a URI for exactly the picture the
user chose, and the app renders it directly in an `Image`.

The same options object also turns on in-picker capture
(`isPhotoTakingSupported: true`), so a user with no suitable photo can shoot one
without leaving the flow.

## Feature checklist

The application must be able to:

- Show a front and a back ID-card slot, each with a placeholder image and an add
  button.
- Open the system photo picker filtered to ID-card recommendations.
- Allow taking a new photo from inside the picker.
- Limit the selection to one image of image MIME type.
- Render the chosen URI in the corresponding slot and hide that slot's add
  button.
- Leave the existing selection untouched when the user cancels the picker.
- Lay out under the status bar and navigation indicator.

## Architecture

Single `entry` HAP with one page - this is the smallest sample in the industry:

| File | Responsibility |
| --- | --- |
| `entryability/EntryAbility.ets` | immersive layout, publishes `topRectHeight` / `bottomRectHeight` into `AppStorage`, loads `pages/Index` |
| `entrybackupability/EntryBackupAbility.ets` | backup stub |
| `pages/Index.ets` | `@Entry`; both card slots, the shared `imagePicker` helper, and the submit button |

Four pieces of state, two per card:

```ts
@State frontIdCard: (ResourceStr | undefined) = $r('app.media.ic_front_id_card');  // placeholder -> picked URI
@State frontAddIcon: (ResourceStr | ImageContent) = $r('app.media.ic_add');        // '+' -> ImageContent.EMPTY
@State backIdCard:  (ResourceStr | undefined) = $r('app.media.ic_back_id_card');
@State backAddIcon: (ResourceStr | ImageContent) = $r('app.media.ic_add');
```

Each slot is a `Stack` of two `Image`s: the card underneath (placeholder, then
the picked photo) and the add icon on top. Hiding the add icon is done by
swapping its source to `ImageContent.EMPTY` rather than by `visibility()` - a
neat trick, but it means the tap target stays in place, which matters for the
cancel path (HW-05-0090).

Flow:

```
add icon onClick
  -> imagePicker(RecommendationType.ID_CARD)
       new photoAccessHelper.PhotoViewPicker()
       PhotoSelectOptions {
         recommendationOptions: { recommendationType },
         isPhotoTakingSupported: true,
         maxSelectNumber: 1,
         MIMEType: PhotoViewMIMETypes.IMAGE_TYPE
       }
       await picker.select(option) -> photoUris[0]
  -> frontIdCard = uri;  frontAddIcon = ImageContent.EMPTY
```

## Implementation steps

1. **Declare no permission.** `PhotoViewPicker` is a system picker that returns a
   read-only URI for the selected item; the sample's `module.json5` has no
   `requestPermissions` block and the document has no 权限说明 section -
   consistent, and correct.
2. **Build the options object.** `recommendationOptions` carries the filter,
   `MIMEType: PhotoViewMIMETypes.IMAGE_TYPE` restricts to images,
   `maxSelectNumber: 1` gives a single result, and `isPhotoTakingSupported: true`
   adds the in-picker camera entry.
3. **Set the recommendation type.**
   `photoAccessHelper.RecommendationType.ID_CARD` for an identity document; the
   enum has other members for other document kinds.
4. **Await the selection and guard the array.** `(await
   picker.select(option)).photoUris` - check `length` before indexing.
5. **Treat cancellation as a normal outcome.** `select` rejects when the user
   dismisses the picker; assign the result **only on success** so an existing
   photo and the add control survive (HW-05-0090).
6. **Render the URI directly.** `Image(this.frontIdCard)` with a fixed box and
   `ImageFit.Cover`; the picker URI is usable as an `Image` source without any
   file I/O.
7. **Hide the add control on success** by swapping its source to
   `ImageContent.EMPTY`, and restore `$r('app.media.ic_add')` if the slot is ever
   cleared.
8. **Publish the insets from `TYPE_SYSTEM`** - this sample does that correctly -
   and release the `avoidAreaChange` subscription in `onWindowStageDestroy`
   (HW-05-0091).

## Verified snippets

All snippets below are copied from the sample project, not from the document.

### The picker with the ID-card recommendation

`RecommendPhoto.zip#entry/src/main/ets/pages/Index.ets`

```ts
import { photoAccessHelper } from '@kit.MediaLibraryKit';

async imagePicker(recommendType: photoAccessHelper.RecommendationType) {
  let result: (ResourceStr | undefined) = undefined;
  let photoPicker = new photoAccessHelper.PhotoViewPicker();
  let option: photoAccessHelper.PhotoSelectOptions = {
    recommendationOptions: { recommendationType: recommendType },
    isPhotoTakingSupported: true,
    maxSelectNumber: 1,
    MIMEType: photoAccessHelper.PhotoViewMIMETypes.IMAGE_TYPE
  };
  try {
    result = (await photoPicker.select(option)).photoUris[0];
  } catch (err) {
    hilog.error(0x0000, 'Index', 'Create photoPicker failed, err.code = ' + err.code);
  }
  return result;
}
```

The options object is the whole feature - passing `recommendationType` is what
makes the gallery open pre-filtered. Corrected result handling:

```ts
async imagePicker(recommendType: photoAccessHelper.RecommendationType): Promise<string | undefined> {
  const photoPicker = new photoAccessHelper.PhotoViewPicker();
  const option: photoAccessHelper.PhotoSelectOptions = {
    recommendationOptions: { recommendationType: recommendType },
    isPhotoTakingSupported: true,
    maxSelectNumber: 1,
    MIMEType: photoAccessHelper.PhotoViewMIMETypes.IMAGE_TYPE
  };
  try {
    const uris = (await photoPicker.select(option)).photoUris;
    return uris.length > 0 ? uris[0] : undefined;
  } catch (err) {
    hilog.error(0x0000, 'Index', 'select failed, code = %{public}d', err.code);
    return undefined;
  }
}
```

### The two card slots - as shipped

`RecommendPhoto.zip#entry/src/main/ets/pages/Index.ets`

```ts
@State frontIdCard: (ResourceStr | undefined) = $r('app.media.ic_front_id_card');
@State backIdCard: (ResourceStr | undefined) = $r('app.media.ic_back_id_card');
@State frontAddIcon: (ResourceStr | ImageContent) = $r('app.media.ic_add');
@State backAddIcon: (ResourceStr | ImageContent) = $r('app.media.ic_add');

Column({ space: 16 }) {
  Stack() {
    Image(this.frontIdCard)
      .width(272)
      .height(164)
      .objectFit(ImageFit.Cover)
      .borderRadius(8);

    Image(this.frontAddIcon)
      .width(36)
      .onClick(async () => {
        this.frontIdCard = await this.imagePicker(photoAccessHelper.RecommendationType.ID_CARD); // HW-05-0090
        if (this.frontIdCard !== undefined) {
          this.frontAddIcon = ImageContent.EMPTY;
        }
      });
  };

  Stack() {
    Image(this.backIdCard)
      .width(272)
      .height(164)
      .objectFit(ImageFit.Cover)
      .borderRadius(8);

    Image(this.backAddIcon)
      .width(36)
      .onClick(async () => {
        this.backIdCard = await this.imagePicker(photoAccessHelper.RecommendationType.ID_CARD);
        if (this.backIdCard !== undefined) {
          this.backAddIcon = ImageContent.EMPTY;
        }
      });
  };
};
```

Corrected click handler - assign only on success:

```ts
.onClick(async () => {
  const picked = await this.imagePicker(photoAccessHelper.RecommendationType.ID_CARD);
  if (picked !== undefined) {
    this.frontIdCard = picked;
    this.frontAddIcon = ImageContent.EMPTY;
  }
});
```

### Insets read from the correct avoid-area type

`RecommendPhoto.zip#entry/src/main/ets/entryability/EntryAbility.ets`

```ts
onWindowStageCreate(windowStage: window.WindowStage): void {
  let windowClass: window.Window = windowStage.getMainWindowSync();
  let isLayoutFullScreen = true;
  windowClass.setWindowLayoutFullScreen(isLayoutFullScreen).then(() => {
    hilog.info(0x0000, 'testTag', 'Succeeded in setting the window layout to full-screen mode.');
  }).catch((err: BusinessError) => {
    hilog.error(0x0000, 'testTag',
      'Failed to set the window layout to full-screen mode. Cause:' + JSON.stringify(err));
  });

  let type = window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR; // 导航条避让
  let avoidArea = windowClass.getWindowAvoidArea(type);
  AppStorage.setOrCreate('bottomRectHeight', avoidArea.bottomRect.height);

  type = window.AvoidAreaType.TYPE_SYSTEM;                   // 状态栏避让
  avoidArea = windowClass.getWindowAvoidArea(type);
  AppStorage.setOrCreate('topRectHeight', avoidArea.topRect.height);

  windowClass.on('avoidAreaChange', (data) => {              // no matching off - HW-05-0091
    if (data.type === window.AvoidAreaType.TYPE_SYSTEM) {
      AppStorage.setOrCreate('topRectHeight', data.area.topRect.height);
    } else if (data.type === window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR) {
      AppStorage.setOrCreate('bottomRectHeight', data.area.bottomRect.height);
    }
  });

  windowStage.loadContent('pages/Index', (err) => {
    if (err.code) {
      hilog.error(DOMAIN, 'testTag', 'Failed to load the content. Cause: %{public}s', JSON.stringify(err));
      return;
    }
  });
}
```

This ability reads the top inset from `TYPE_SYSTEM`, which is the correct choice
for a status bar - unlike OFFICE-09 (HW-05-0053) and OFFICE-15 (HW-05-0086),
which use `TYPE_CUTOUT` for the same purpose. The page consumes the values in
`px`, converting at the point of use:

```ts
@StorageLink('topRectHeight') topRectHeight: number = 1;
@StorageLink('bottomRectHeight') bottomRectHeight: number = 1;

.padding({
  top: this.getUIContext().px2vp(this.topRectHeight),
  bottom: this.getUIContext().px2vp(this.bottomRectHeight),
  left: 16,
  right: 16
});
```

## Permissions & config

`RecommendPhoto.zip#entry/src/main/module.json5` declares **no
`requestPermissions` block**, and none is needed:

- `PhotoViewPicker` runs outside the application and returns a URI only for what
  the user selected, so `ohos.permission.READ_IMAGEVIDEO` is not required.
- `isPhotoTakingSupported: true` adds a camera entry **inside the picker**, which
  is still the system's camera - `ohos.permission.CAMERA` is not required either.
  Compare OFFICE-05 (HW-05-0022) and OFFICE-07 (HW-05-0040), where a camera
  permission was declared for a system picker that does not need it.

The document has no 权限说明 section, which matches - verified consistent.

`build-profile.json5` pins the SDK to `6.0.0(20)`, and the document's project
tree matches the shipped ZIP exactly, including `entrybackupability`.

## Constraints

- **Versions.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later.
- **The recommendation is a filter, not a guarantee.** The picker surfaces the
  photos it believes match the requested type; the user can still pick anything
  else, and the app must not assume the returned image is an ID card.
- **The returned URI is read-only.** The Picker guide states: "The permission on
  the URIs returned by select() is read-only. File data can be read based on the
  URI. Note that the URI cannot be directly used in the Picker callback to open a
  file. You need to define a global variable to save the URI and use a button to
  open the file." Rendering it in an `Image`, as this sample does, is fine; opening
  it with `fileIo` inside the callback is not.
- **Cancellation is a rejection.** `select` rejects rather than resolving with an
  empty result when the user dismisses the picker, so the failure path is on the
  `catch`, not on `photoUris.length`.
- **`ImageContent.EMPTY` renders an empty image, it does not remove the
  component.** The add icon keeps its layout box and its tap target after being
  "hidden" this way.
- **One image per slot.** `maxSelectNumber: 1` and `photoUris[0]`; the front and
  back slots each run their own independent pick.
- **Nothing is uploaded or persisted.** The submit button has no handler and the
  URIs live only in `@State`, so the selection is lost on restart.

## Pitfalls

- **`this.frontIdCard = await this.imagePicker(...)` is incorrect.** The helper
  returns `undefined` when the user cancels, so the assignment erases whatever was
  there - the placeholder on the first attempt, the chosen photo afterwards. And
  because `frontAddIcon` was already swapped to `ImageContent.EMPTY` on the
  earlier success, the user is left with a blank card and no visible control to
  try again. Assign inside the `!== undefined` guard, and guard `photoUris[0]`
  against an empty array too. (HW-05-0090)
- **Subscribing to `avoidAreaChange` without a matching `off` is incorrect** -
  release it in `onWindowStageDestroy`. (HW-05-0091)
- **The document's step-3 snippet discards the picker result, which is
  incorrect** for a step titled "display the selected image URI in the Image
  component": `await this.imagePicker(...)` with no assignment target leaves
  `this.frontIdCard` untouched, so the `Image` shown underneath it is bound to
  something the snippet never sets. (HW-05-0092)

## References

Reference pages used to verify this card:

- `documentation/harmonyos-guides/05_media/photoaccesshelper-photoviewpicker.md` -
  the `PhotoViewPicker.select` flow with its `.catch()` on rejection, and the
  read-only-URI caveat about not opening the file inside the picker callback.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/photoaccesshelper-photoviewpicker
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-image.md` -
  `Image(src: PixelMap | ResourceStr | DrawableDescriptor | ImageContent)` and the
  `ImageContent12+` enum with its single `EMPTY` member.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-image
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-window.md` -
  `setWindowLayoutFullScreen`, `getWindowAvoidArea`, `on`/`off('avoidAreaChange')`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-window
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-e.md` -
  `AvoidAreaType.TYPE_SYSTEM` as the status-bar area.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-e
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-photoviewpicker.md`
  and `arkts-apis-photoaccesshelper-e.md` - the reference pages the document
  names for `PhotoViewPicker` and `RecommendationType`; both are stubs in this
  repository, so the API details above were taken from the Picker guide instead
  and `RecommendationType.ID_CARD` could not be checked against a member list.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-photoviewpicker
