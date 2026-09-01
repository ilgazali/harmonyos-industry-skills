---
id: SOCIAL-10
title: Avatar crop and profile edit - a circular Canvas mask over a Matrix4-transformed image, persisted as Base64 in Preferences
industry: 14_social_communication
doc: huawei_industry_tree/14_social_communication/docs/10_personal_info_upload.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/personal_info_upload-0000002290265961
sample: huawei_industry_tree/14_social_communication/downloads/PersonalInfoUpload.zip
kits: ["@kit.AbilityKit", "@kit.ArkData", "@kit.ArkTS", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.ImageKit", "@kit.MediaLibraryKit", "@kit.PerformanceAnalysisKit"]
apis: [fileIo, hilog, image, matrix4, photoAccessHelper, preferences, util, window]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-14-0021, HW-14-0022, HW-14-0023, HW-14-0087]
status: verified-with-fixes
---

## When to use

Load this card for the **"edit my profile" screen**: an avatar the user
replaces from the gallery and crops to a circle, plus a stack of one-field
sub-pages for name, gender, birthday, phone and signature. It is the same
screen in a chat app, a forum, a delivery app and an internal directory.

Two mechanisms are worth taking away. The first is the **crop viewport**: the
image sits under a `Matrix4` transform driven by pan and pinch gestures, a
`Canvas` painted over it punches a transparent circle out of a dark mask with
`globalCompositeOperation = 'destination-out'`, and the crop is computed by
inverting the same transform into source-pixel coordinates. No image is
resampled until Save. The second is the **field-page pattern**: the profile page
owns the model, each sub-page is a `NavDestination` registered in the router
map, and the push carries an `onPop` callback that re-reads the store when the
sub-page returns.

**Read `HW-14-0021`, `HW-14-0022` and `HW-14-0023` before adopting this one.**
The crop viewport is good code. The persistence layer around it is built on a
detached `UIContext`, the edit pages start empty and overwrite whatever was
stored, and the back gesture is disabled in every destination in the app. Take
the geometry, rewrite the plumbing.

## Feature checklist

- A profile page shows the avatar plus five rows: name, gender, birthday, phone
  number, signature - each with the stored value and a chevron.
- 更换头像 (change avatar) opens the system photo picker, limited to a single
  PNG.
- The picked image opens in a full-screen crop page: a dark mask with a
  transparent circular viewport and a white outline.
- One finger pans the image, two fingers pinch to zoom, a double tap toggles
  between 1x and 2x.
- The image can never be dragged so that the viewport shows empty space - it
  springs back to cover the circle.
- Save crops the source pixels under the viewport, encodes the result as PNG
  Base64 and stores it; the profile page shows the new avatar on return.
- Each field row pushes a single-field editor with its own Save button.
- Values survive a restart of the app.

## Architecture

One `entry` module. A profile page, six `NavDestination` editors, and a
singleton store.

```
entry/src/main/ets
├── constants/StyleConstants.ets         every literal size/colour used by the pages
├── entryability/EntryAbility.ets        full screen, avoid areas -> AppStorage (raw px)
├── entrybackupability/EntryBackupAbility.ets
├── pages/MainPage.ets                   @Entry: padding from the avoid areas, hosts Home
├── pages/Home.ets                       the profile page: Navigation host + the five rows
├── utils/LocalDataManager.ets           singleton over one Preferences store
├── utils/PersonalInfoModel.ets          PersonalInfo, dateToString, pixelToBase64, uriParam
├── utils/Preference.ets                 a second, entirely unused Preferences helper
└── views/
    ├── ChangeName.ets                   TextInput + Save
    ├── ChangeGender.ets                 two rows with a tick
    ├── ChangeDate.ets                   collapsible DatePicker
    ├── ChangePhone.ets                  TextInput + Save
    ├── ChangeSignature.ets              TextInput + Save
    └── ImageCrop.ets                    CropView + CropModel + the crop page (565 lines)
```

The documented 工程目录 matches the zip. (The industry tree-mismatch
systematic `HW-14-0001` covers four other social samples, not this one.)

`utils/Preference.ets` is dead code: a `PreferencesClass` with `setToken` /
`getToken` that nothing imports. The document lists it as
"preferences类型定义页" without saying it is unused.

**The design decision worth copying** is the split inside `ImageCrop.ets`:
`CropModel` is a plain class holding the geometry (source size, component size,
frame size, scale, offsets) and the two methods that do arithmetic on it
(`checkImageAdapt`'s counterpart `crop()`, and `getFrameHeight()`); `CropView`
is the component that turns gestures into model updates and the model into a
`Matrix4`. Because the model knows the component size (captured from
`Image.onComplete`), `crop()` can map the on-screen viewport back to source
pixels without the component being present. That is what makes the cropper
reusable - and the reason it is worth lifting this file out of the sample
wholesale.

**The structural decision worth avoiding** is `LocalDataManager` as a
zero-argument singleton. It has no context to build a `Preferences` instance
with, so it manufactures one (`new UIContext()`) - and that is the defect in
`HW-14-0021`. A store that needs a context should take it as an argument.

## Implementation steps

1. **Pick with `PhotoViewPicker`**, `maxSelectNumber = 1` and a
   `MimeTypeFilter`. No permission is needed: the picker runs out of process
   and hands back a URI the app may read.
2. **Push the crop page by name with the URI as the parameter**, and pass an
   `onPop` callback that re-reads the store - the crop page writes the store,
   not the caller's state.
3. **Let `Image` size itself with `ImageFit.Contain`, and capture the four sizes
   from `onComplete`** (`width`, `height`, `componentWidth`, `componentHeight`).
   Everything downstream is arithmetic on those.
4. **Apply pan and pinch through one `Matrix4`**, translate then scale, rebuilt
   from the model on every update. Divide the pan delta by the current scale so
   dragging tracks the finger at any zoom.
5. **Clamp after every gesture**, not during: `onActionEnd` runs
   `checkImageAdapt`, which grows the scale if the image no longer covers the
   viewport and then pulls the offsets back inside the frame.
6. **Paint the mask with a `Canvas` on top and `globalCompositeOperation =
   'destination-out'`**, then switch back to `'source-over'` for the outline.
   Set `.enabled(false)` so the canvas does not eat the gestures.
7. **Crop by inverting the transform**: the viewport's top-left in source-pixel
   space is `(frameX - showX) / totalScale`, and the region size is the frame
   size divided by the same factor.
8. **Deep-copy the PixelMap before `cropSync`** via `readPixelsToBuffer`, and
   remember that the buffer comes out BGRA while `createPixelMap` wants RGBA.
9. **Build the store with the real ability context** - never `new UIContext()`
   (`HW-14-0021`).
10. **Prefill every editor from the store in `aboutToAppear`, and let Save run
    more than once** (`HW-14-0022`).
11. **Do not return `true` from `onBackPressed` unless you actually handled the
    press** (`HW-14-0023`).

## Verified snippets

All snippets are from `PersonalInfoUpload.zip`. Corrected forms are marked.

**The store — `entry/src/main/ets/utils/LocalDataManager.ets`** (corrected, see `HW-14-0021`)

```typescript
import { preferences } from '@kit.ArkData';
import { Context } from '@kit.AbilityKit';
import { PersonalInfo } from './PersonalInfoModel';

export class LocalDataManager {
  private static localDataManager: LocalDataManager;
  private PERSONAL_INFO: string = 'personalInfo';
  private personalInfo: PersonalInfo = {
    name: $r('app.string.small_luck'),
    gender: '男',
    date: new Date(),
    phoneNo: $r('app.string.phone_number'),
    personalSignature: $r('app.string.no_fill'),
    pm: ''
  };
  private preferencesInstance: preferences.Preferences;

  // FIX: the sample builds the store from `new UIContext().getHostContext()`, which is undefined
  private constructor(context: Context) {
    this.preferencesInstance = preferences.getPreferencesSync(context, { name: this.PERSONAL_INFO });
  }

  static instance(context: Context) {
    if (!LocalDataManager.localDataManager) {
      LocalDataManager.localDataManager = new LocalDataManager(context);
    }
    return LocalDataManager.localDataManager;
  }

  upload() {
    this.preferencesInstance.putSync(this.PERSONAL_INFO, this.personalInfo);
    this.preferencesInstance.flushSync();
  }

  queryInfo() {
    let personalInfo = this.preferencesInstance.getSync(this.PERSONAL_INFO, this.personalInfo) as PersonalInfo;
    personalInfo.date = new Date(personalInfo.date);
    this.personalInfo = personalInfo;
    return this.personalInfo;
  }
}
```

Call it from the page, where a context exists:

```typescript
private localDataManager?: LocalDataManager;

aboutToAppear(): void {
  this.localDataManager = LocalDataManager.instance(this.getUIContext().getHostContext() as Context);
  this.personalInfo = this.localDataManager.queryInfo();
}
```

**`new UIContext()` is not an accessor, it is a constructor** - it builds a
`UIContext` bound to nothing, and `getHostContext()` on it returns `undefined`.
`preferences.getPreferencesSync(undefined, ...)` cannot resolve an application
directory, so the first `LocalDataManager.instance()` call - which happens in a
field initializer on `Home`, `ChangeName` and four more pages - takes the whole
persistence layer down with it. Every symptom the sample shows (values that do
not survive a restart, an avatar that reverts) traces back to this one line.

The date round-trip is the other thing to keep: `Preferences` serialises the
object, and a `Date` comes back as a string, so `queryInfo` re-wraps it with
`new Date(personalInfo.date)` before returning. Without that line
`dateToString` would throw on `getFullYear`.

**The circular mask — `entry/src/main/ets/views/ImageCrop.ets`** (as shipped)

```typescript
Canvas(this.context)
  .width('100%')
  .height('100%')
  .backgroundColor(Color.Transparent)
  .onReady(() => {
    if (this.context == null) {
      return;
    }
    let height = this.context.height;
    let width = this.context.width;
    this.context.fillStyle = this.model.maskColor;          // '#AA000000'
    this.context.fillRect(0, 0, width, height);

    let centerX = width / 2;
    let centerY = height / 2;

    // 把中间的取景框透出来 - punch the viewport out of the mask
    this.context.globalCompositeOperation = 'destination-out';
    this.context.fillStyle = 'white';
    let frameWidthInVp = this.getUIContext().px2vp(this.model.frameWidth);
    let frameHeightInVp = this.getUIContext().px2vp(this.model.getFrameHeight());
    this.context.beginPath();
    this.context.arc(centerX, centerY, this.getUIContext().px2vp(this.model.frameWidth / 2), 0, 2 * Math.PI);
    this.context.fill();

    // 设置综合操作模式为源覆盖 - back to normal painting for the outline
    this.context.globalCompositeOperation = 'source-over';
    this.context.strokeStyle = this.model.strokeColor;
    let radius = Math.min(frameWidthInVp, frameHeightInVp) / 2;
    this.context.beginPath();
    this.context.arc(centerX, centerY, radius, 0, 2 * Math.PI);
    this.context.closePath();
    this.context.lineWidth = 1;
    this.context.stroke();
  })
  .enabled(false);
```

**`destination-out` is the whole trick.** Fill the canvas with the mask colour,
switch the composite mode so the next fill *removes* alpha instead of adding
it, draw the circle, and the mask now has a hole exactly where the viewport is.
The alternative - four rectangles around a circle, or a clip path on the image -
either cannot make a round hole or forces the image itself to be clipped, which
would break the drag-outside-the-circle preview.

Two easy-to-miss lines: switching back to `'source-over'` before stroking, or
the outline would erase instead of draw; and `.enabled(false)`, without which
the canvas sits on top of the image and swallows every pan and pinch.

`frameWidth` is stored in **px** (1000) and converted with `px2vp` for
painting, because canvas coordinates are vp while the crop arithmetic works in
source pixels. Keeping the frame in px is what lets `crop()` use it directly.

**Inverting the transform to a crop region — same file** (as shipped)

```typescript
public async crop(): Promise<image.PixelMap> {
  if (this.imageWidth === 0 || this.imageHeight === 0) {
    throw new Error('The image is not loaded');
  }
  // 图片适配控件的时候也进行了缩放 - the ImageFit.Contain scale
  let widthScale = this.componentWidth / this.imageWidth;
  let heightScale = this.componentHeight / this.imageHeight;
  let adaptScale = Math.min(widthScale, heightScale);

  // 经过两次缩放后，图片的实际显示大小 - fit scale times gesture scale
  let totalScale = adaptScale * this.scale;
  let showWidth = this.imageWidth * totalScale;
  let showHeight = this.imageHeight * totalScale;
  let imageX = (this.componentWidth - showWidth) / 2;
  let imageY = (this.componentHeight - showHeight) / 2;

  let frameX = (this.componentWidth - this.frameWidth) / 2;
  let frameY = (this.componentHeight - this.getFrameHeight()) / 2;

  let showX = imageX + this.offsetX * this.scale;
  let showY = imageY + this.offsetY * this.scale;
  let x = (frameX - showX) / totalScale;               // viewport top-left in source pixels
  let y = (frameY - showY) / totalScale;

  let file = fileIo.openSync(this.src, fileIo.OpenMode.READ_ONLY);
  let imageSource: image.ImageSource = image.createImageSource(file.fd);
  let pm = await imageSource.createPixelMap({ editable: true,
    desiredPixelFormat: image.PixelMapFormat.BGRA_8888 });
  let cp = await this.copyPixelMap(pm);
  pm.release();
  let region: image.Region =
    { x: x, y: y, size: { width: this.frameWidth / totalScale, height: this.getFrameHeight() / totalScale } };
  cp.cropSync(region);
  fileIo.closeSync(file);
  return cp;
}
```

**The image is scaled twice and the crop has to undo both.** `adaptScale` is
what `ImageFit.Contain` applied to fit the picture into the component;
`this.scale` is what the pinch applied on top. Their product maps source pixels
to on-screen vp, so dividing the on-screen distance between the frame and the
image origin by `totalScale` gives the offset in source pixels - the crop is
therefore taken at full source resolution, not at preview resolution, which is
why zooming out and saving still yields a sharp avatar.

`copyPixelMap` exists because `cropSync` mutates in place and the decoded
PixelMap is backed by the file. Note the format asymmetry the sample flags in
its own comments: `readPixelsToBuffer` writes **BGRA_8888**, but
`image.createPixelMap` is told **RGBA_8888** - a channel swap that goes
unnoticed on a grey test photo and shows up as blue-tinted skin on a real one.
Decode as `RGBA_8888` if you keep this copy step.

**A field editor — `entry/src/main/ets/views/ChangeName.ets`** (corrected, see `HW-14-0022`, `HW-14-0023`)

```typescript
@Component
struct ChangeName {
  pageInfos: NavPathStack | null = null;
  @State name: string = '';
  private localDataManager?: LocalDataManager;

  aboutToAppear(): void {                                  // FIX: absent in the sample
    this.localDataManager = LocalDataManager.instance(this.getUIContext().getHostContext() as Context);
    const stored = this.localDataManager.queryInfo().name;
    this.name = typeof stored === 'string' ? stored : '';
  }

  build() {
    NavDestination() {
      Column() {
        Button($r('app.string.save'))
          .onClick(() => {
            this.localDataManager?.changeName(this.name);  // FIX: no one-shot isSave latch
          });

        TextInput({ placeholder: $r('app.string.input_name'), text: $$this.name })
          .width(StyleConstants.FULL)
          .height(StyleConstants.HEIGHT_56)
          .borderRadius(StyleConstants.BORDER_RADIUS_16)
          .backgroundColor(Color.White);
      }
      .backgroundColor($r('app.color.dark_white'));
    }
    .hideTitleBar(true, true)
    .onReady((context: NavDestinationContext) => {
      this.pageInfos = context.pathStack;
    });                                                    // FIX: no onBackPressed(() => true)
  }
}
```

**Two independent defects live in the six lines this replaces.** The field
starts at `''` and Save writes it, so opening the name page and pressing Save
without typing erases the stored name - the page is a blind write, not an edit.
And `isSave` is a one-way latch (`if (this.isSave) { this.isSave = false; ... }`),
so after a single Save the button greys out and a correction needs a full exit
and re-entry. `ChangeDate` is the worst case: it starts at `new Date()`, so a
bare Save resets the birthday to today.

`$$this.name` is a two-way binding, so `TextInput` writes straight back into
the state - which is exactly why the initial value must come from the store.

The removed `.onBackPressed(() => true)` is `HW-14-0023`: **every**
`NavDestination` in this app (all five editors plus `ImageCrop`) returns `true`
without doing anything, and `MainPage.onBackPress` does the same. Returning
`true` means "handled, do not pop" - so the system back gesture is dead
app-wide, every sub-page can only be left through the on-screen arrow, and the
app itself cannot be backed out of. Return `true` only when you have actually
consumed the press (as `WebPage` in `SOCIAL-12` does, where it walks the web
history first).

## Permissions & config

**None declared.** `PhotoViewPicker` runs in the system's picker process and
returns a URI the app is granted read access to for the session - so no
`READ_IMAGEVIDEO` and no `WRITE_IMAGEVIDEO`. If you replace the picker with
`photoAccessHelper.getPhotoAccessHelper` and query the album directly, that
changes.

`main_pages.json` lists all seven pages **and** `route_map.json` maps six of
them to builder functions. The editors are consequently declared twice, and
each editor struct carries a redundant `@Entry` decorator even though it is
only ever reached as a `NavDestination`. Registering them as router-map
destinations alone is enough.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- The picker is restricted to `image/png` by `MimeTypeFilter`; a JPEG chosen in
  the gallery is not selectable. Widen `mimeTypeArray` for real use - the crop
  path itself is format-agnostic because it goes through `createImageSource`.
- The avatar is stored as a Base64 **PNG string inside Preferences**. That is
  fine for a demo and wrong at scale: `Preferences` is a key-value store meant
  for small settings, and a 1000x1000 PNG runs to hundreds of kilobytes that
  are re-read synchronously on every page entry. Store the file and keep the
  path.
- `frameWidth` is fixed at 1000 px with `frameRatio` 1, set in
  `ImageCrop.onReady`. A non-square viewport works (`getFrameHeight()` divides
  by the ratio) but nothing in the UI exposes it.
- `EntryAbility` writes **raw px** avoid-area values into `AppStorage`; the
  pages convert with `px2vp` at the point of use. `SOCIAL-09` uses the opposite
  convention.
- `utils/Preference.ets` is unused. Delete it rather than wiring it up.

## Pitfalls

- **`HW-14-0021`** (B/medium, probable): `LocalDataManager` builds its
  `Preferences` instance from `new UIContext().getHostContext()`, which is
  `undefined`, in a field initializer - the whole profile persistence fails at
  the first `instance()` call. Fix: take the ability context as a constructor
  argument and pass it from the page.
- **`HW-14-0022`** (B/medium, confirmed): the editor pages never prefill from
  the store, so a bare Save wipes the stored value (`ChangeDate` resets the
  birthday to today), and the one-shot `isSave` latch prevents correcting it in
  place. Fix: load current values in `aboutToAppear` and let Save repeat.
- **`HW-14-0023`** (B/medium, confirmed): `onBackPress`/`onBackPressed` return
  `true` in `MainPage` and in all six `NavDestination`s without acting, so the
  system back gesture does nothing anywhere in the app. Fix: remove the blanket
  `true` returns.
- **`HW-14-0001`** (E/low, confirmed; systematic): the industry's documented
  project trees do not always match the zips. This sample's does - but check
  before trusting a sibling doc's file list.
- Unfiled, worth knowing: `copyPixelMap` reads BGRA and recreates as RGBA (the
  sample's own comment admits it); `pixelToBase64` swallows every failure into
  an empty string, which `updateImage` then persists as "no avatar"; and
  `ImageCrop.handleFileSelection` is dead code.

## References

- `huawei_industry_tree/14_social_communication/docs/10_personal_info_upload.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/personal_info_upload-0000002290265961
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-f.md` - `PhotoViewPicker`, `PhotoSelectOptions`, `MimeTypeFilter`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-f
- `documentation/harmonyos-references/02_application-framework/js-apis-data-preferences.md` - `getPreferencesSync` and the context it needs
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-data-preferences
- `documentation/harmonyos-references/02_application-framework/js-components-canvas-canvasrenderingcontext2d.md` - `globalCompositeOperation`, `arc`, `fillRect`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-components-canvas-canvasrenderingcontext2d
- `documentation/harmonyos-references/02_application-framework/js-apis-matrix4.md` - `Matrix4.identity().translate().scale()`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-matrix4
- `documentation/harmonyos-references/04_media/arkts-apis-image-pixelmap.md` - `cropSync`, `readPixelsToBuffer`, pixel formats
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-image-pixelmap
- `documentation/harmonyos-references/04_media/arkts-apis-image-imagepacker.md` - `packToData` and `PackingOption`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-image-imagepacker
- `documentation/harmonyos-references/02_application-framework/js-apis-util.md` - `Base64Helper`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-util
- `documentation/harmonyos-guides/03_application-framework/arkts-navigation-navdestination.md` - `onReady`, `onBackPressed` and what returning `true` means
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-navigation-navdestination
