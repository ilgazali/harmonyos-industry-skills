---
id: MEDIA-19
title: Moving-photo carousel - picking live photos from the gallery and auto-playing the centre one in a vertical Swiper
industry: 13_media_entertainment
doc: huawei_industry_tree/13_media_entertainment/docs/19_dynamic_photo.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/dynamic_photo-0000002321583145
sample: huawei_industry_tree/13_media_entertainment/downloads/DynamicPhoto.zip
kits: ["@kit.AbilityKit", "@kit.ArkData", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.MediaLibraryKit", "@kit.PerformanceAnalysisKit"]
apis: [dataSharePredicates, hilog, photoAccessHelper, window]
permissions: [ohos.permission.READ_IMAGEVIDEO]
min_api: 20
modules: [entry (entry)]
findings: [HW-13-0048, HW-13-0032, HW-13-0002, HW-13-0098]
status: verified-with-fixes
---

## When to use

Load this card when you need to **display a user's moving photos (live photos)
inside your own UI** rather than handing them to the system gallery - a comment
thread that plays the attached live photo, a news card, a memories strip, a
profile header.

The pipeline has three stages and each one is a different API family: the
**picker** returns URIs, `photoAccessHelper` turns a URI into a `PhotoAsset`,
and `MediaAssetManager.requestMovingPhoto` turns a `PhotoAsset` into the
`MovingPhoto` object that `MovingPhotoView` can render. Nothing shorter works:
`MovingPhotoView` takes a `MovingPhoto`, not a URI.

The presentation idea worth lifting is the **carousel that plays only its
centre item**. Three cards are visible; the middle one auto-plays and repeats,
the two flanking it are dimmed by a gradient overlay and stay still. That is a
cheap and readable way to make "which one is live" obvious without any extra
chrome, and it applies equally to video previews, animated stickers or GIF
strips.

**Read `HW-13-0002` before adopting the middle stage.** The sample's
`getAssets` query needs a runtime permission it never requests, so on a real
device the six photos never load at all - the presentation layer is the only
part of this sample that is actually exercised.

## Feature checklist

- A black full-screen page with a 正在播放列表 (now playing list) heading.
- A single 选择动态照片 (choose moving photos) button when nothing is loaded.
- The button opens the system photo picker filtered to moving photos only.
- Exactly six must be selected; any other count raises a toast and aborts.
- Each selected URI is resolved to a `MovingPhoto` and rendered in a vertical
  `Swiper` showing three cards at a time.
- Every card carries a fixed title (第一页 … 第六页) above the photo.
- The centre card auto-plays its motion on a loop, muted, for the first 600 ms
  of the clip; the other two are frozen and dimmed under a gradient overlay.
- The carousel advances by itself every 10 ms of dwell with a 50 ms linear
  transition, looping endlessly.

## Architecture

One `entry` module, two ArkUI files. There is no model layer: the six
`MovingPhoto` objects live in one `@State` array inside the view.

```
entry/src/main/ets
├── constants/StyleConstants.ets   sizes, margins and the literal true/false/0/600 used as options
├── entryability/EntryAbility.ets  full-screen layout, avoid areas -> AppStorage
├── entrybackupability/EntryBackupAbility.ets
├── pages/MainPage.ets             @Entry, 33 lines: black background + avoid-area padding
└── views/MovingPhoto.ets          the whole feature: picker, asset resolution, Swiper
```

The documented 工程目录 matches the zip.

`MainPage` does one thing and does it in the recommended way: it reads the two
avoid-area heights with `@StorageProp` (typed default `0`) and converts with
`px2vp` at the point of use, then hosts `MovingPhoto()`. Compare `MEDIA-20`,
which reads only the top inset and hardcodes the rest.

**The design decision worth avoiding** is the module-level
`let uiContext = new UIContext();` at the top of `views/MovingPhoto.ets`. Three
things in this file hang off it - the toast (`getPromptAction()`), the ability
context (`getHostContext()`) and the `PhotoAccessHelper` built from that
context - and all three are field initialisers, so they run at component
construction from an instance that is attached to no window
(`HW-13-0032`). `UIContext` must come from `this.getUIContext()` inside the
component; a bare-constructed one has no host. Four media samples in this
industry make the same mistake, so treat it as a pattern to recognise, not an
isolated slip.

The structure *is* right in one respect worth keeping: `selectPhoto` takes the
destination array as a parameter (`selectPhoto(this.srcList)`) instead of
writing `this.srcList` directly, which is what lets the `onDataPrepared`
callback push into it without capturing `this` across an async boundary. It
also means the array identity is the contract, which is precisely why the
ordering problem in `HW-13-0048` is invisible from the call site.

## Implementation steps

1. **Filter the picker to moving photos** with
   `PhotoSelectOptions.MIMEType = PhotoViewMIMETypes.MOVING_PHOTO_IMAGE_TYPE`
   and cap the count with `maxSelectNumber`.
2. **Validate the returned count before doing any work** - the picker can
   return fewer than the maximum, and the fixed six-title `Swiper` cannot cope.
3. **Get the `UIContext` from the component** (`this.getUIContext()`), not from
   `new UIContext()` (`HW-13-0032`).
4. **Resolve each URI to a `PhotoAsset`** with a `DataSharePredicates`
   `equalTo(PhotoKeys.URI, uri)` query - and note that this query needs
   `ohos.permission.READ_IMAGEVIDEO` granted at runtime, which the sample never
   requests (`HW-13-0002`).
5. **Request the `MovingPhoto`** through `MediaAssetManager.requestMovingPhoto`
   with `DeliveryMode.FAST_MODE`, pushing the result from `onDataPrepared`.
6. **Only reveal the `Swiper` after every photo is prepared** - not at the
   start of the last iteration (`HW-13-0048`).
7. **Drive playback from a centre index**, recomputed in `Swiper.onChange` as
   `(currentIndex + 1) % count` because `displayCount(3)` reports the *first*
   of the three visible cards.
8. **Dim the non-centre cards** with an `overlay` builder rather than an
   opacity on the view itself, so the moving photo keeps its own rendering.

## Verified snippets

All snippets are from `DynamicPhoto.zip`. Corrected forms are marked.

**The three-stage resolution — `entry/src/main/ets/views/MovingPhoto.ets`** (corrected, see `HW-13-0048`, `HW-13-0032`)

```typescript
import { MovingPhotoView, photoAccessHelper, MovingPhotoViewAttribute } from '@kit.MediaLibraryKit';
import { dataSharePredicates } from '@kit.ArkData';

const MAX_PHOTO_NUM = 6;

async selectPhoto(srcList: photoAccessHelper.MovingPhoto[]) {
  try {
    const uiContext = this.getUIContext();                   // FIX: module-level new UIContext()
    const context = uiContext.getHostContext();
    const phAccessHelper = photoAccessHelper.getPhotoAccessHelper(context);

    // picker selects the moving-photo URIs.
    let photoSelectOptions = new photoAccessHelper.PhotoSelectOptions();
    photoSelectOptions.MIMEType = photoAccessHelper.PhotoViewMIMETypes.MOVING_PHOTO_IMAGE_TYPE;
    photoSelectOptions.maxSelectNumber = MAX_PHOTO_NUM;
    let photoViewPicker = new photoAccessHelper.PhotoViewPicker();
    let photoSelectResult = await photoViewPicker.select(photoSelectOptions);
    let uris = photoSelectResult.photoUris;
    if (uris.length !== MAX_PHOTO_NUM) {
      uiContext.getPromptAction().showToast({
        message: $r('app.string.select_six_dynamic_images')
      });
      return;
    }
    for (let i = 0; i < uris.length; i++) {
      // resolve the URI to its PhotoAsset.
      let predicates: dataSharePredicates.DataSharePredicates = new dataSharePredicates.DataSharePredicates();
      predicates.equalTo(photoAccessHelper.PhotoKeys.URI, uris[i]);
      let fetchOption: photoAccessHelper.FetchOptions = {
        fetchColumns: [],
        predicates: predicates
      };
      let fetchResult: photoAccessHelper.FetchResult<photoAccessHelper.PhotoAsset> =
        await phAccessHelper.getAssets(fetchOption);
      let photoAsset: photoAccessHelper.PhotoAsset = await fetchResult.getFirstObject();
      // turn the PhotoAsset into a MovingPhoto object.
      await new Promise<void>((resolve) => {
        photoAccessHelper.MediaAssetManager.requestMovingPhoto(context, photoAsset, {
          deliveryMode: photoAccessHelper.DeliveryMode.FAST_MODE
        }, {
          async onDataPrepared(movingPhoto: photoAccessHelper.MovingPhoto) {
            if (movingPhoto !== undefined) {
              srcList[i] = movingPhoto;                      // FIX: shipped code pushes
            }
            resolve();                                       // FIX: await the callback, not the request
          }
        });
      });
    }
    this.isChosen = true;                                    // FIX: shipped code sets this
                                                             //      inside the last iteration, first
  } catch (err) {
    hilog.error(DOMAIN, 'MovingPhoto', `request moving photo failed with error: ${err.code}, ${err.message}`);
  }
}
```

**`requestMovingPhoto` resolves when the request is accepted, not when the data
is ready** - `onDataPrepared` is the completion signal, and `FAST_MODE` means
"give me whatever quality is available soonest", which makes the callback
timing genuinely variable. Three consequences follow, and the shipped code hits
all three.

First, `await`ing the request and then continuing the loop does not guarantee
the photos land in selection order; `srcList[i] = ...` instead of
`srcList.push(...)` makes the index explicit, and wrapping the request in a
promise that the callback resolves makes the loop sequential. Second,
`this.isChosen = true` sits at the *top* of the sixth iteration, before that
iteration's `getAssets` and `requestMovingPhoto` have even started, so the
`Swiper` builds while `srcList[5]` is `undefined` - `MovingPhotoView` receives
an undefined `movingPhoto` on first render. Moving the flag past the loop is
the fix. Third, `fetchColumns: []` is deliberate: the query only needs the
asset handle, not its columns, and `getFirstObject` is safe because a URI
equality predicate matches at most one asset.

The permission problem sits underneath all of this. `getAssets` requires
`ohos.permission.READ_IMAGEVIDEO`, which `module.json5` declares but no code
ever requests with `requestPermissionsFromUser`, so on a device the query
throws and the `catch` swallows it into a log line (`HW-13-0002`). Two escapes
exist: request the permission properly, or - better for a picker-driven flow -
drop the `photoAccessHelper` query entirely, since the picker's returned URI is
already granted to your app and the file APIs need no permission at all.

**The centre-plays carousel — same file** (as shipped)

```typescript
Swiper() {
  ForEach(this.titleList, (item: string, index: number) => {
    Column({ space: StyleConstants.SPACE_10 }) {
      Row() {
        Text(this.titleList[index])
          .fontSize(StyleConstants.FONT_SIZE_14)
          .fontColor(index === this.centerIndex ? Color.White : $r('app.color.light_white'));
      }.width(StyleConstants.FULL);

      MovingPhotoView({
        movingPhoto: this.srcList[index]
      })
        .width(StyleConstants.FULL)
        .muted(StyleConstants.TRUE)
        .autoPlay(this.centerIndex === index ? StyleConstants.TRUE : StyleConstants.FALSE)
        .repeatPlay(this.centerIndex === index ? StyleConstants.TRUE : StyleConstants.FALSE)
        .autoPlayPeriod(StyleConstants.ZERO, StyleConstants.SIX_HUNDRED)
        .objectFit(ImageFit.Contain)
        .overlay(this.centerIndex === index ? undefined : this.overlayBuilder());
    }
  }, (item: string) => item);
}
.displayCount(StyleConstants.THREE, StyleConstants.FALSE)
.loop(StyleConstants.TRUE)
.vertical(StyleConstants.TRUE)
.indicator(StyleConstants.FALSE)
.interval(StyleConstants.INTERVAL_10)
.duration(StyleConstants.DURATION_50)
.curve(Curve.Linear)
.onChange((currentIndex: number) => {
  this.centerIndex = (currentIndex + 1) % this.titleList.length;
});
```

**Four attributes carry the effect.** `displayCount(3, false)` puts three cards
on screen with no swipe-by-group, so the strip scrolls one card at a time and
there is always a genuine middle. `onChange` reports the index of the *first*
visible card, hence the `+ 1` to reach the centre; the modulo is required
because `loop(true)` wraps past the end. `autoPlay` and `repeatPlay` are bound
to that comparison rather than set once, so motion follows the centre position
automatically - no imperative start/stop calls anywhere in the file.
`autoPlayPeriod(0, 600)` clips the loop to the first 600 ms, which is what
keeps six simultaneous live photos from looking chaotic.

`muted(true)` is not optional in a carousel: moving photos carry audio, and
three of them are alive across a transition.

**The dimming overlay — same file** (as shipped)

```typescript
@Builder
overlayBuilder() {
  Stack()
    .height(StyleConstants.FULL)
    .width(StyleConstants.FULL)
    .linearGradient({
      direction: GradientDirection.Bottom,
      colors: [[$r('app.color.zero_white'), 0.0], [Color.White, 1.0]]
    })
    .blendMode(BlendMode.DST_IN, BlendApplyType.OFFSCREEN)
    .hitTestBehavior(HitTestMode.None);
}
```

This is a **mask, not a tint**. `BlendMode.DST_IN` keeps the destination (the
photo) only where the source (this gradient) is opaque, so a gradient running
from transparent white at the top to solid white at the bottom fades the card
out towards its top edge. `BlendApplyType.OFFSCREEN` is what scopes the blend
to this subtree instead of compositing against whatever is behind the page -
without it the result depends on the black page background.
`hitTestBehavior(HitTestMode.None)` lets taps and swipes pass straight through
to the `Swiper`. Applying it through `.overlay()` rather than as a sibling in a
`Stack` means the non-centre cards need no layout of their own.

## Permissions & config

```json5
"requestPermissions": [
  {
    "name": 'ohos.permission.READ_IMAGEVIDEO',
    "reason": '$string:reason',
    "usedScene": {
      "abilities": ["EntryAbility"],
      "when": 'always'
    },
  }
]
```

- `READ_IMAGEVIDEO` is **user_grant and restricted**: declaring it is not
  enough, it needs a runtime `requestPermissionsFromUser`, and a restricted
  permission additionally needs an ACL entry to be grantable at all. The sample
  does neither (`HW-13-0002`).
- `when: 'always'` is questionable for a foreground picker flow; `'inuse'` is
  the right scope for reading the gallery while the user is looking at the app.
- If the `getAssets` stage is dropped in favour of the picker URI - the fix
  this finding recommends - the permission can be removed from `module.json5`
  entirely.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`,
  so the constraint text and the build profile agree here.
- **Exactly six photos, always.** `titleList` is six hardcoded string resources
  and the `Swiper` iterates it, so the carousel's length is fixed at build
  time; the picker's result is validated against `MAX_PHOTO_NUM` and any other
  count aborts with a toast. Making this dynamic means iterating `srcList` and
  generating titles.
- There is no re-pick path: once `isChosen` flips true the button is gone and
  the only way back is restarting the page.
- Moving photos are a device-gallery concept. On a device with none, the picker
  returns an empty selection and the sample toasts - it ships no bundled
  fallback asset, so the feature cannot be exercised without real live photos.
- The `ForEach` key generator is typed `(item: string) => item` over a
  `Resource[]`, so it stringifies a resource object for the key. It happens to
  work because the six entries are distinct, but the declared type is wrong.
  (Not a filed finding; noted so it is not copied.)
- `MovingPhotoViewAttribute` is imported and never used.

## Pitfalls

- **`HW-13-0048`** (B/medium, confirmed): `this.isChosen = true` is set at the
  *start* of the last loop iteration, before that iteration's `getAssets` and
  `requestMovingPhoto` run, so the `Swiper` renders with `srcList[5]`
  undefined; and because `onDataPrepared` pushes asynchronously, the array's
  order does not have to match the selection order either. Fix: set the flag
  after the loop, index into `srcList` instead of pushing, and await each
  callback.
- **`HW-13-0032`** (B/medium, probable): `let uiContext = new UIContext();` at
  module scope - a detached instance whose `getHostContext()` has no host -
  feeds `getPromptAction()`, the ability context and
  `getPhotoAccessHelper()`. Three other media samples (`AVPlayerAudio`,
  `SplashPage`, `M3U8Download`) do the same, and doc 11 reproduces the code.
  Fix: take the context from `this.getUIContext()` inside the component.
- **`HW-13-0002`** (B/medium, confirmed): `getAssets` and
  `MediaAssetManager.requestMovingPhoto` require
  `ohos.permission.READ_IMAGEVIDEO`, which is declared in `module.json5` but
  never requested at runtime, so the six selected photos never load. The same
  defect appears in `MEDIA-03` (`GetVideoInfo`) and `MEDIA-20` (`MergeVideo`),
  and in two other industries. Fix: operate on the picker-returned URI
  directly, or request the permission before querying.

## References

- `documentation/harmonyos-references/04_media/ohos-multimedia-movingphotoview.md` - `MovingPhotoView`, `autoPlay`, `repeatPlay`, `autoPlayPeriod`, `muted`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ohos-multimedia-movingphotoview
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-movingphoto.md` - the `MovingPhoto` object
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-movingphoto
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-mediaassetmanager.md` - `requestMovingPhoto`, `DeliveryMode`, `onDataPrepared`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-mediaassetmanager
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-photoviewpicker.md` - `PhotoSelectOptions`, `PhotoViewMIMETypes.MOVING_PHOTO_IMAGE_TYPE`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-photoviewpicker
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-f.md` - `getPhotoAccessHelper`, `getAssets` and its permission requirement
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-f
- `documentation/harmonyos-guides/04_system/restricted-permissions.md` - `ohos.permission.READ_IMAGEVIDEO` as a restricted user_grant permission
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/restricted-permissions
- `documentation/harmonyos-guides/04_system/request-user-authorization.md` - the runtime request flow the sample omits
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/request-user-authorization
- `documentation/harmonyos-references/02_application-framework/ts-container-swiper.md` - `displayCount`, `loop`, `vertical`, `onChange` index semantics
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-swiper
- `MEDIA-03` and `MEDIA-20` - the two other instances of the same `getAssets` permission defect
