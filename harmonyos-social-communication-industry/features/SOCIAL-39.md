---
id: SOCIAL-39
title: Send the most recent photo - RecentPhotoComponent as a one-tap hint over the chat toolbar
industry: 14_social_communication
doc: huawei_industry_tree/14_social_communication/docs/39_recent_image.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/recent_image-0000002355555898
sample: huawei_industry_tree/14_social_communication/downloads/ChatRecentImage.zip
kits: ["@kit.AbilityKit", "@kit.ArkTS", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.ImageKit", "@kit.MediaLibraryKit", "@kit.PerformanceAnalysisKit"]
apis: [RecentPhotoComponent, RecentPhotoOptions, PhotoSource, BaseItemInfo, onRecentPhotoCheckInfo, onRecentPhotoClick, onRecentPhotoCheckResult, "UIContext.createAnimator", AnimatorResult, ListScroller, scrollEdge, KeyboardAvoidMode, getFocusController, NavPathStack, "image.createImagePacker", packToData, buffer, fs, hilog]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-14-0017, HW-14-0018, HW-14-0087, HW-14-0089, HW-14-0090]
status: verified-with-fixes
---

## When to use

**Load this card when the chat toolbar should offer the photo the user just
took** - the small floating card that appears above the "+" menu saying
"最近图片", showing one thumbnail, one tap away from being sent. It is the
shortcut that removes the picker from the most common image-sending case.

The mechanism is `RecentPhotoComponent`, a **security component** from
MediaLibraryKit. Like `PhotoViewPicker` and `SaveButton`, it renders in a
separate process, so the app never touches the gallery and needs no gallery
permission; it learns only what the user actually taps, and only because the
click callback returns `true`. The generalisation is the important part: any
"suggest the newest item from a protected store" affordance - a recent
document, a recent recording - should be built the same way, with the system
component deciding what to show and the app deciding what to do with the one
item that comes back.

The second half of the card is the **frame animator** that opens the toolbar.
`createAnimator` drives the menu height frame by frame and scrolls the message
list to the bottom on every frame, so the conversation stays pinned to the
last message while the panel grows. That pairing - an animator that also
scrolls - is worth copying whenever a bottom panel eats into a list.

**Read `HW-14-0017` before adopting the send path.** The success toast fires
before the compression that produces the message.

## Feature checklist

- The chat page shows a seeded two-message conversation and a bottom input row.
- Tapping the "+" icon expands a four-entry function menu with a 200ms
  ease-in-out animation; tapping again collapses it.
- Focusing the text input collapses the menu; opening the menu stops the
  editing session first.
- While the menu is open, a 104x182 card floats above it showing the newest
  gallery image, labelled 最近图片.
- The card is present only if the gallery actually has a matching photo, and
  appears about 200ms after the check reports one.
- Collapsing the menu cancels the pending reveal and hides the card again.
- Tapping the thumbnail collapses the menu, clears input focus, and pushes a
  send page carrying the photo's `BaseItemInfo`.
- The send page previews the image, offers an 原图 (original) checkbox, and a
  send button.
- Unchecked, the image is compressed to JPEG at quality 5 and appended to the
  conversation as a base64 data URI; checked, the raw URI is appended.
- The list scrolls to the newest image whenever the picture array changes.

## Architecture

One `entry` module, five source files. Two pages inside one `Navigation`,
routed by name through a `routerMap`.

```
entry/src/main/ets
├── common/Constants.ets        every layout number and colour in the sample
├── entryability/EntryAbility.ets  full screen, avoid areas (in vp) -> AppStorage
├── pages/
│   ├── ChatPage.ets            @Entry: Navigation host, the toolbar animator, RecentPhotoComponent
│   └── SendPage.ets            NavDestination: preview, 原图 toggle, send
└── utils/
    ├── ImageUtils.ets          compressImage: uri -> packToData(ArrayBuffer)
    └── Logger.ets              hilog wrapper
```

The documented 工程目录 matches the zip. One documentation detail does not
match: the doc's snippet types the check-info callback as
`(recentPhotoExists: boolean, info: RecentPhotoInfo)`, while the shipped code
declares it with the single `recentPhotoExists` parameter and never uses the
info object. The shipped one-parameter form is what compiles against the
`RecentPhotoCheckInfoCallback` in this SDK; treat the doc's second parameter as
illustrative.

**The design decision worth copying** is that the recent-photo card is
rendered *inside* the animated menu container but positioned out of it:

```typescript
.position({ top: -198, right: 16 })
.visibility(this.hasRecentPhoto ? Visibility.Visible : Visibility.Hidden)
```

It is a child of the menu `Column`, so it appears and disappears with the
menu and inherits its lifetime, but the negative `top` lifts it clear above
the toolbar where a hint belongs. Making it a sibling in a page-level `Stack`
would have meant duplicating the show/hide condition; making it a real popup
would have meant anchoring maths. This is the cheap correct answer.

The complementary decision is the double gate on visibility. `if (this.showMenu)`
controls whether the card exists; `hasRecentPhoto` controls whether it is
visible. The component must be *mounted* before it can run its own gallery
check and call back - so it is mounted invisible, and only revealed once the
callback says there is something to show.

## Implementation steps

1. **Configure `RecentPhotoOptions` in `aboutToAppear`**, before the component
   can mount: `MIMEType` (`IMAGE_TYPE` to exclude video), `period` in seconds
   (`0` = no time limit) and `photoSource` (`ALL`, `CAMERA` or `SCREENSHOT`).
2. **Bind the three callbacks as fields, not inline arrows.** The component
   takes them by reference; declaring them as typed private fields keeps the
   `this` binding and satisfies the callback types.
3. **Mount the component invisible** and flip `hasRecentPhoto` from
   `onRecentPhotoCheckInfo`, on a short `setTimeout` so the reveal lands after
   the menu animation rather than during it.
4. **Return `true` from `onRecentPhotoClick`** - this is not a formality. The
   returned boolean is what grants the app read access to the returned URI;
   return `false` and the URI is unusable.
5. **Clear focus before navigating** with
   `getUIContext().getFocusController().clearFocus()`, otherwise the soft
   keyboard survives the page push.
6. **Drive the toolbar height with an `AnimatorResult`**, and call
   `scrollEdge(Edge.End)` from `onFrame` so the list keeps its last item
   visible while the panel grows.
7. **Cancel the pending reveal on collapse**: `clearTimeout(this.timeoutID)`
   and reset `hasRecentPhoto`, or a fast open-close leaves the card showing
   over a closed menu.
8. **Pass the whole `BaseItemInfo` as the navigation param** and read it in
   `onReady`; `recentPhotoInfo.uri` is the only field the sample uses, but the
   info object also carries size and mediaType for a size warning.
9. **Await the compression before reporting success** and give it a `catch`
   (`HW-14-0017`), and label the data URI with the MIME you actually packed.
10. **Key the message `ForEach` by something unique** - the bare URI collides
    when the same photo is sent twice (`HW-14-0018`).

## Verified snippets

All snippets are from `ChatRecentImage.zip`. Corrected forms are marked.

**Options and callbacks — `entry/src/main/ets/pages/ChatPage.ets`** (as shipped)

```typescript
private recentPhotoOptions: RecentPhotoOptions = new RecentPhotoOptions();
private recentPhotoCheckResultCallback: RecentPhotoCheckResultCallback =
  () => this.onRecentPhotoCheckResult();
private recentPhotoClickCallback: RecentPhotoClickCallback =
  (recentPhotoInfo: BaseItemInfo): boolean => this.onRecentPhotoClick(recentPhotoInfo);
private recentPhotoCheckInfoCallback: RecentPhotoCheckInfoCallback =
  (recentPhotoExists: boolean) => this.onRecentPhotoCheckInfo(recentPhotoExists);

aboutToAppear(): void {
  this.getUIContext().setKeyboardAvoidMode(KeyboardAvoidMode.RESIZE);

  // 设置数据类型， IMAGE_VIDEO_TYPE：图片和视频（默认值）  IMAGE_TYPE：图片   VIDEO_TYPE：视频
  this.recentPhotoOptions.MIMEType = photoAccessHelper.PhotoViewMIMETypes.IMAGE_TYPE;
  // 设置最近图片的时间范围，单位（秒）， 0表示所有时间。
  this.recentPhotoOptions.period = 0;
  // 设置资源的来源 ALL：所有 CAMERA：相机  SCREENSHOT：截图
  this.recentPhotoOptions.photoSource = PhotoSource.ALL;
  // ... animator setup
}

// 返回值为true时，才能获取uri的权限
private onRecentPhotoClick(recentPhotoInfo: BaseItemInfo): boolean {
  if (!recentPhotoInfo) {
    LOGGER.info('onRecentPhotoClick false');
    return false;
  }
  this.changeMenuStatus();
  this.getUIContext().getFocusController().clearFocus();
  this.pageInfos.pushPath({ name: 'sendPage', param: recentPhotoInfo });
  return true;
}

private onRecentPhotoCheckInfo(recentPhotoExists: boolean): void {
  if (!recentPhotoExists) {
    LOGGER.info('not exist recent photo');
    return;
  }
  this.timeoutID = setTimeout(() => {
    this.hasRecentPhoto = true;
  }, 200);
  LOGGER.info('exist recent photo');
}
```

**The `return true` is the permission grant, and the sample's own comment says
so** (返回值为true时，才能获取uri的权限 - "only when the return value is true
can you obtain the uri's permission"). `RecentPhotoComponent` runs
out-of-process; the app receives a URI it has no standing right to read, and
returning `true` from the click callback is what converts the user's tap into
a read authorisation for that one asset. Returning `false` - as the guard does
when the info object is missing - correctly declines it.

The three `RecentPhotoOptions` fields are the whole query. `period: 0` means
"no time window at all", which is generous for a demo; a real chat usually
wants 60 or 300 seconds so the hint offers a photo the user *just* took rather
than whatever is newest in a gallery from last year. `photoSource: ALL` mixes
camera shots and screenshots; `CAMERA` is the narrower and usually better
choice for a "send what you just shot" hint.

`onRecentPhotoCheckResult` is bound but empty in this sample - it is the hook
for reacting to the query completing regardless of outcome.

**Mounting invisible, revealing late — same file** (as shipped)

```typescript
if (this.showMenu) {
  Column() {
    Text($r('app.string.recent_image'))
      .fontSize(8)
      .fontColor('rgba(0, 0, 0, 0.6)')
      .lineHeight(9);
    RecentPhotoComponent({
      recentPhotoOptions: this.recentPhotoOptions,
      onRecentPhotoCheckResult: this.recentPhotoCheckResultCallback,
      onRecentPhotoClick: this.recentPhotoClickCallback,
      onRecentPhotoCheckInfo: this.recentPhotoCheckInfoCallback
    })
      .width(80)
      .height(150);
  }
  .width(104)
  .height(182)
  .borderRadius(8)
  .backgroundColor(Color.White)
  .padding(10)
  .position({ top: -198, right: 16 })
  .visibility(this.hasRecentPhoto ? Visibility.Visible : Visibility.Hidden)
  .justifyContent(FlexAlign.SpaceBetween);
}
```

**`Visibility.Hidden`, not `Visibility.None`, is the correct choice here.**
`None` would remove the node from the layout and the component would never
mount, never query the gallery, and never call back - the card could then only
ever be hidden. `Hidden` keeps it laid out and running while invisible, which
is exactly what a self-checking security component needs.

The 200ms `setTimeout` in the callback matches the animator's 200ms duration:
the card is revealed as the toolbar finishes opening rather than popping in
mid-slide. `changeMenuStatus` clears that timeout on collapse, which is the
detail that keeps a fast double-tap from leaving the card stranded.

**The toolbar animator — same file** (as shipped)

```typescript
this.menuAnimator = this.getUIContext()?.createAnimator({
  duration: 200,
  easing: 'ease-in-out',
  delay: 0,
  fill: 'forwards',
  direction: 'normal',
  iterations: 1,
  begin: 52 + this.bottomRectHeight,   //动画插值起点
  end: 147 + this.bottomRectHeight     //动画插值终点
});

this.menuAnimator.onFrame = (value: number) => {
  this.menuHeight = value;
  this.listScroller.scrollEdge(Edge.End);
};

private changeMenuStatus() {
  if (this.showMenu) {
    this.menuAnimator?.reset({ /* ... */ begin: 147 + this.bottomRectHeight,
                                        end: 52 + this.bottomRectHeight });
  } else {
    this.menuAnimator?.reset({ /* ... */ begin: 52 + this.bottomRectHeight,
                                        end: 147 + this.bottomRectHeight });
  }
  this.showMenu = !this.showMenu;
  this.menuAnimator?.play();
  if (!this.showMenu) {
    if (this.timeoutID) {
      clearTimeout(this.timeoutID);
      this.timeoutID = 0;
    }
    this.hasRecentPhoto = false;
  }
}
```

**`fill: 'forwards'` plus `reset` is what makes one animator do both
directions.** The animator holds its end value after finishing, so the menu
stays open; reversing means swapping `begin` and `end` and calling `play()`
again, not running the same animation backwards. Both endpoints include
`bottomRectHeight` because the toolbar has to clear the navigation indicator.

`scrollEdge(Edge.End)` inside `onFrame` is the reason the conversation does
not appear to slide up out of view as the panel grows: the list is re-pinned
to its bottom edge on every interpolated frame. The same handler is reused by
`listScrollBottom`, the `@Watch` on the `pictures` array, so a newly sent image
scrolls into view too.

**The send path — `entry/src/main/ets/pages/SendPage.ets`**
(corrected, see `HW-14-0017`)

```typescript
Button($r('app.string.send'))
  .onClick(async () => {
    try {
      if (!this.isChecked) {
        await this.selectAndCompressPicture();     // FIX: shipped code does not await
      } else {
        if (this.imageUri) {
          this.pictures.push(this.imageUri[0]);
        }
      }
      this.pageInfos.pop();
      this.getUIContext().getPromptAction().showToast({ message: $r('app.string.send_success') });
    } catch (err) {
      // FIX: shipped code has no catch - a failed compression was an unhandled rejection
      this.getUIContext().getPromptAction().showToast({ message: $r('app.string.file_compression_failed') });
    }
  });

async selectAndCompressPicture(): Promise<void> {
  let pictureUriArr: Array<string> = this.imageUri;
  await Promise.all(pictureUriArr.map(async (uri) => {         // FIX: forEach dropped every promise
    const data: ArrayBuffer = await ImageUtils.compressImage(uri, 'image/jpeg', this.quality);
    let base64Str = buffer.from(data).toString('base64');
    let resultBase64Str = 'data:image/jpeg;base64,' + base64Str;  // FIX: was data:image/png
    this.pictures.push(resultBase64Str);
  }));
}
```

**Three independent defects sit in eight lines, and they are the same three
found in `SendOriginalImage` (`SOCIAL-08`).** The `forEach` + `.then` shape
drops the promise, so `selectAndCompressPicture` resolves before any
compression has finished; the caller then pops the page and toasts
"发送成功" while the image may still be encoding - or may have failed, since
there is no `.catch` anywhere in the chain, making a failure an unhandled
rejection and a message that silently never appears. And the payload is packed
as `image/jpeg` but labelled `data:image/png;base64,` - harmless for
`Image()`, which sniffs the bytes, and wrong for anything that trusts the
prefix, such as a real transport or a web view.

`ImageUtils.compressImage` itself is sound: it opens the URI read-only, reads
the bytes into an `ArrayBuffer`, hands them to `image.createImageSource` and
returns `imagePackerApi.packToData(imageSource, { format, quality })`, closing
the source file in a `finally`. Quality `5` on a JPEG is extremely aggressive -
that is the demo making the 原图 checkbox visibly meaningful, not a sensible
default.

**The message list — `entry/src/main/ets/pages/ChatPage.ets`**
(corrected, see `HW-14-0018`)

```typescript
List({ space: 24, initialIndex: Constants.ZERO, scroller: this.listScroller }) {
  this.messageBuilder();
  ForEach(this.pictures, (uri: string, index: number) => {
    ListItem() {
      this.imageBuilder(uri, index);
    };
  }, (item: string, index: number) => `${index}_${item.slice(0, 32)}`)  // FIX: key was `item`
}
.contentEndOffset(10)
.listDirection(Axis.Vertical)
.alignListItem(ListItemAlign.End)
.edgeEffect(EdgeEffect.Spring, { alwaysEnabled: true })
.layoutWeight(1);
```

**A content-only key breaks on the most ordinary user action there is: sending
the same photo twice.** The shipped generator is `(item: string) => item`, so
two identical URIs - or two identical base64 strings - produce the same key,
ArkUI's diff treats the second as already present, and the second send does not
render. Including the index makes the key unique; slicing the value keeps the
key short, which matters when the "value" is a multi-megabyte data URI.

`alignListItem(ListItemAlign.End)` is what right-aligns outgoing images
without a per-item wrapper, and `contentEndOffset(10)` leaves a gap under the
last bubble so `scrollEdge(Edge.End)` does not butt the newest message against
the toolbar.

## Permissions & config

**None declared.** `module.json5` has an empty permission surface, and that is
the headline: `RecentPhotoComponent` is a security component that reads the
gallery in its own process and hands back a single authorised URI when the
click callback returns `true`. No `ohos.permission.READ_IMAGEVIDEO` anywhere.

```json5
"pages": "$profile:main_pages",
"routerMap": "$profile:route_map",
"deviceTypes": ["phone", "tablet", "2in1"]
```

`EntryAbility` sets `COLOR_MODE_LIGHT`, calls `setWindowLayoutFullScreen(true)`
and publishes both avoid areas to `AppStorage` - **already converted with
`px2vp`**, unlike most samples in this industry, which store px and convert at
the point of use. Both pages therefore consume `topRectHeight` /
`bottomRectHeight` as plain vp numbers. Pick one convention and keep it: mixing
the two across pages is how avoid-area bugs happen. The `avoidAreaChange`
listener is never unregistered.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- `RecentPhotoComponent` shows exactly one item and gives the app nothing until
  the user taps it. There is no API to read the newest photo silently, by
  design.
- The component must remain mounted and laid out to run its check - use
  `Visibility.Hidden`, never `Visibility.None`, to suppress it.
- `period: 0` in this sample means the "recent" photo may be arbitrarily old.
  Set a real window for production.
- The send page always operates on `imageUri[0]`; multi-select is not part of
  this flow (see `SOCIAL-08` for the picker-based multi-image path).
- Compression quality is hardcoded to 5 and the format to `image/jpeg`; there
  is no size threshold, so a small image is recompressed too.
- The conversation is not persisted: `pictures` is a page-level `@Provide`
  array seeded empty, and the two text messages are static resources.

## Pitfalls

- **`HW-14-0017`** (B/medium, confirmed, systematic - `ChatRecentImage` is the
  second instance after `SendOriginalImage`): the send button pops the page and
  toasts 发送成功 before `selectAndCompressPicture` completes, because the
  method's `forEach` + `.then` drops the promise and the caller does not await
  it; no `.catch` exists, so a failed compression is an unhandled rejection and
  the message silently never appears. The same code labels JPEG bytes as
  `data:image/png;base64,`. Fix: `await` the compression inside a `try`/`catch`,
  toast on completion, and use `data:image/jpeg`.
- **`HW-14-0018`** (B/medium, confirmed, systematic across six chat samples):
  the message `ForEach` keys on the bare URI/base64 string
  (`(item: string) => item`), so sending the same image twice produces
  duplicate keys and the second send is not rendered. Fix: include the index or
  a message id in the key.

## References

- `huawei_industry_tree/14_social_communication/docs/39_recent_image.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/recent_image-0000002355555898
- `documentation/harmonyos-references/04_media/ohos-file-recentphotocomponent.md` - `RecentPhotoComponent`, `RecentPhotoOptions`, the three callbacks and the `true` return contract
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ohos-file-recentphotocomponent
- `documentation/harmonyos-guides/03_application-framework/arkts-animator.md` - `createAnimator`, `onFrame`, `reset`, `fill: 'forwards'`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-animator
- `documentation/harmonyos-references/02_application-framework/ts-container-list.md` - `ListScroller`, `scrollEdge`, `contentEndOffset`, `alignListItem`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-list
- `documentation/harmonyos-guides/03_application-framework/arkts-rendering-control-foreach.md` - key generators and why duplicates break diffing
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-rendering-control-foreach
- `documentation/harmonyos-guides/03_application-framework/arkts-common-events-focus-event.md` - `getFocusController().clearFocus()`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-common-events-focus-event
- `documentation/harmonyos-references/04_media/js-apis-image.md` - `createImageSource`, `packToData`, `PackingOption`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-image
- `SOCIAL-08` - the picker-based original-image send path, where both systematic findings were first raised
