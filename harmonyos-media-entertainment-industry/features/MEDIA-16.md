---
id: MEDIA-16
title: Short-video comment sheet - a DIALOG NavDestination that slides up over the shrinking player
industry: 13_media_entertainment
doc: huawei_industry_tree/13_media_entertainment/docs/16_short_video_review.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/short_video_review-0000002286277826
sample: huawei_industry_tree/13_media_entertainment/downloads/ShortVideoReview.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.IMEKit", "@kit.PerformanceAnalysisKit"]
apis: [display, hilog, inputMethod, window]
min_api: 20
modules: [entry (entry)]
findings: [HW-13-0043, HW-13-0044, HW-13-0098]
status: verified-with-fixes
---

## When to use

Load this card for the **comment sheet over a full-screen video**: tap the
speech-bubble icon, the video shrinks to a thumbnail at the top, a rounded
panel slides up from the bottom carrying the comment list, and a tap on the
input raises a second, smaller sheet with the keyboard. It is the interaction
every short-video app ships, and the same shape covers a product page's review
drawer, a map app's place sheet, or a photo's comment tray.

The technique worth taking is the **two-layer sheet**. The comment list is a
`NavDestination` with `mode: NavDestinationMode.DIALOG`, pushed onto a
`NavPathStack` - so it gets a real back-stack entry, a system back gesture,
and a `pop` that can return a result - while the keyboard row is a
`CustomDialog` opened *on top of it*. Splitting them means the list can scroll
under a persistent input bar without the keyboard's layout changes disturbing
the list, and the input sheet can be dismissed independently.

`NavDestinationMode.DIALOG` is the load-bearing option: it makes the
destination transparent and leaves the page underneath mounted and visible,
which is what allows the video thumbnail to keep playing behind the panel.
Without it you get an opaque page and have to rebuild the player.

**Read both findings before shipping.** The keyboard listener is registered
per open and never removed, and a submitted comment leaves the placeholder
resource sitting in the input as real text.

## Feature checklist

- A full-screen looping video with the built-in controls hidden; tapping it
  toggles play/pause and reveals a centred play icon when paused.
- A right-hand rail of avatar, follow, like, comment and share icons; all but
  comment answer with a "demo only" toast.
- The comment icon stops the video and pushes the comment destination.
- The comment panel slides up over 400 ms from the bottom edge, with 32 vp
  rounded top corners over a dimmed background.
- The video reappears as a 144x253 vp thumbnail near the top, still looping.
- The list carries two canned comment threads plus everything the user has
  sent this session.
- Tapping the bottom input opens a second sheet holding a focused text field,
  an emoji button and an image button, with the keyboard already up.
- Submitting appends the comment to the list.
- The keyboard sheet closes itself when the keyboard is dismissed.
- Tapping the dimmed area or the close icon pops back to the video.

## Architecture

One `entry` module. The comment destination is registered in a router map, not
imported by the page that pushes it.

```
entry/src/main/ets
├── common/Constants.ets                numeric constants (NUMBER_0 … NUMBER_500)
├── component
│   ├── ShortVideo.ets                  the Navigation root: video, icon rail, the push
│   ├── VideoDes.ets                    author, caption and timestamp over the bottom-left
│   └── DialogFirst.ets                 the comment NavDestination + the CommentDialog @CustomDialog
├── entryability/EntryAbility.ets       avoid areas -> AppStorage
├── entrybackupability/EntryBackupAbility.ets
├── model
│   ├── BasicDataSource.ets             IDataSource with listener bookkeeping
│   └── VideoData.ets                   VideoData, User, @Observed Comment, CommentDataSource
└── pages/Index.ets                     @Entry, 30 lines: one ShortVideo filling the screen
```

The documented tree matches the zip.

```json5
// entry/src/main/resources/base/profile/route_map.json
{
  "routerMap": [
    {
      "name": "DialogFirst",
      "pageSourceFile": "src/main/ets/component/DialogFirst.ets",
      "buildFunction": "DialogFirstBuilder"
    }
  ]
}
```

**The design decision worth copying** is that routing goes through this map
and `"routerMap": "$profile:route_map"` in `module.json5`, so `ShortVideo`
never imports `DialogFirst`. It pushes a **name**. That is what keeps the comment
sheet lazily loaded - the destination's module is not pulled into the video
page's bundle - and it is the mechanism that would let the same sheet be
pushed from a feed page, a profile page or a deep link without any of them
knowing where it lives.

The `NavPathStack` itself is shared by `@Provide` on `ShortVideo` and
`@Consume` in `DialogFirst`, which is how the destination pops itself without
a callback chain.

**The decision worth avoiding** is that `DialogFirst` creates its own
`VideoController` and its own `Video` component pointing at the same
`$r('app.media.video3')`. There is no shared player: the sheet starts a second
decode of the same file from the beginning while the first is stopped. The
visual illusion is that the video shrank; the reality is a restart. For a
demo that is fine, for a product it is a wasted decode and a jump in playback
position - hoist the player above the navigation and pass the controller down.

## Implementation steps

1. **Wrap the video page in `Navigation(this.pageInfos)`** with
   `hideTitleBar(true)` and `mode(NavigationMode.Stack)`, and `@Provide` the
   stack so destinations can pop themselves.
2. **Register the destination in `route_map.json`** and point
   `module.json5` at it with `"routerMap": "$profile:route_map"`. Export a
   `@Builder` function whose name matches `buildFunction`.
3. **Push by name with a pop callback**:
   `pushPathByName('DialogFirst', param, (popInfo) => {...})`. The callback is
   where the video should be restarted - the sample only builds an unused log
   string, so the player stays stopped after the sheet closes.
4. **Declare the destination `NavDestinationMode.DIALOG`** so the video page
   underneath stays mounted and visible through the transparent background.
5. **Animate the panel with `transition`, not `animateTo`**:
   `TransitionEffect.translate({ x: 0, y: '100%', z: 0 }).animation({ duration:
   400, curve: Curve.Linear })` on the panel `Column`, gated by a `@State
   show` flag so the effect runs on appear and on disappear.
6. **Delay the `pop` by the transition duration** when dismissing by tap, so
   the slide-down is visible before the destination unmounts.
7. **Shrink the video in `onStart`**, not in `aboutToAppear` - the size and
   position are only meaningful once the surface has content.
8. **Make the panel's own input non-focusable** (`focusable(false)`) and open
   the `CustomDialog` from its `onClick`. It is a button that looks like a text
   field; the real field lives in the dialog.
9. **Give the dialog's field `defaultFocus(true)`** so the keyboard rises with
   the sheet instead of needing a second tap.
10. **Track the keyboard with `window.on('keyboardHeightChange')`, and remove
    the listener in `aboutToDisappear`** - the sample registers a new one on
    every open and never calls `off` (`HW-13-0044`).
11. **Reset the input to `''` after submit**, never to the placeholder
    resource (`HW-13-0043`).

## Verified snippets

All snippets are from `ShortVideoReview.zip`. Corrected forms are marked.

**Pushing the sheet — `entry/src/main/ets/component/ShortVideo.ets`** (corrected: the pop callback)

```typescript
@Provide('NavPathStack') pageInfos: NavPathStack = new NavPathStack();
@State isPlay: boolean = true;
controller: VideoController = new VideoController();

// the comment icon in the right-hand rail
Column() {
  Image($r('app.media.shortvideo_new_icon'))
    .height(56)
    .width(40)
    .margin({ bottom: 16 });
}
.onClick(() => {
  this.controller.stop();
  this.isPlay = false;
  let test = $r('app.string.shortvideo_test');
  this.pageInfos.pushPathByName('DialogFirst', test, (popInfo) => {
    // FIX: the sample only builds an unused message string here
    this.controller.start();          // the video was stopped on the way in
    this.isPlay = true;
  });
});

// the whole page
Navigation(this.pageInfos) {
  Stack({ alignContent: Alignment.TopStart }) { /* video, rail, VideoDes */ }
    .height('100%');
}
.mode(NavigationMode.Stack)
.titleMode(NavigationTitleMode.Mini)
.hideTitleBar(true)
.height('100%');
```

**`pushPathByName` takes three things and the third is the interesting one.**
The name is resolved through the router map, the parameter is handed to the
destination, and the callback fires when that destination pops - carrying
`popInfo.result`, whatever the sheet passed to `pop()`. That is the channel a
comment sheet needs: "the user posted, refresh the count". The sample pops
with `$r('app.string.dialogfirst_test3')` and the callback formats it into a
string that is immediately discarded, so nothing resumes the player. Put the
restart there.

`this.controller.stop()` rather than `pause()` on the way in is a deliberate
consequence of the sheet spinning up its own player: stopping releases the
surface. If you hoist the player instead, this becomes a `pause()` and the
pop callback becomes a `start()` from the same position.

**The sliding panel — `entry/src/main/ets/component/DialogFirst.ets`** (as shipped)

```typescript
build() {
  NavDestination() {
    Stack() {
      if (this.show) {
        Column()                                   // the dimmed backdrop; tapping it dismisses
          .width('100%')
          .height('100%')
          .backgroundColor(Color.Black)
          .onClick(() => {
            inputMethod.getController().stopInputSession();
            this.show = false;                     // plays the exit transition
            setTimeout(() => {
              this.pageInfos.pop($r('app.string.dialogfirst_test3'));
            }, 400);                               // ... then unmount, after it finishes
          });

        Video({ src: $r('app.media.video3'), controller: this.controller })
          .width(this.videoWidth)
          .height(this.videoHeight)
          .position({ top: this.videoPositionTop, left: this.videoPositionLeft })
          .loop(true)
          .autoPlay(true)
          .controls(false)
          .onStart(() => {
            this.videoHeight = 253;                // shrink to a thumbnail once it is actually playing
            this.videoWidth = 144;
            this.videoPositionTop = 36;
            this.videoPositionLeft = this.screenWidth / 2 - 72;
          });

        Column() { /* title row, List of comments, the fake input */ }
          .borderRadius({ topLeft: 32, topRight: 32 })
          .width('100%')
          .height('66%')
          .position({ x: '0%', y: '35%' })
          .backgroundColor('rgba(26, 26, 26, 0.9)')
          .transition(TransitionEffect.translate({ x: 0, y: '100%', z: 0 })
            .animation({ duration: 400, curve: Curve.Linear }));
      }
    };
  }
  .hideTitleBar(true)
  .mode(NavDestinationMode.DIALOG);
}
```

**Three things make this a sheet rather than a page.**
`NavDestinationMode.DIALOG` keeps the destination transparent and the page
below mounted. The `if (this.show)` wrapper is what gives `transition` both an
appear and a disappear to animate - a `TransitionEffect` only runs when the
component enters or leaves the tree, so a destination that is unmounted by
`pop()` directly would vanish without an exit animation. And the `setTimeout`
of exactly the transition's 400 ms is the manual join between the two: flip
`show` to false, let the slide-down play, then pop.

`stopInputSession()` before dismissing is not optional. The keyboard is owned
by the input method, not by the dialog, and popping the destination with the
keyboard up leaves it floating over the video page.

The `y: '100%'` translate is relative to the component's own height, so the
panel travels exactly its own height regardless of screen size - the one part
of this geometry that is resolution-independent. The `'35%'` position and the
144x253 thumbnail are not.

**The keyboard sheet — same file** (corrected, see `HW-13-0044`)

```typescript
@CustomDialog
struct CommentDialog {
  @Consume rightData: Array<ResourceStr>;
  @Link textInputText: ResourceStr;
  controller?: CustomDialogController;
  @State marginBottom: number = 0;
  uiContext = this.getUIContext();
  private win?: window.Window;                                     // FIX: keep the handle
  private onKeyboardHeightChange = (height: number) => {           // FIX: keep the callback
    if (height) {
      this.marginBottom = -16;
    } else {
      this.controller?.close();          // keyboard dismissed -> the sheet has no reason to stay
    }
  };

  aboutToAppear(): void {
    window.getLastWindow(this.uiContext.getHostContext()).then((win) => {
      this.win = win;
      win.on('keyboardHeightChange', this.onKeyboardHeightChange);
    });
  }

  aboutToDisappear(): void {
    this.win?.off('keyboardHeightChange', this.onKeyboardHeightChange);   // FIX: absent in the sample
  }
}
```

**Closing the sheet when the keyboard closes is the right coupling.** The
dialog exists only to host a focused field; once the user dismisses the
keyboard there is nothing left to show, so `height === 0` is treated as "the
user is done". The `-16` bottom margin in the other branch pulls the sheet
down under the keyboard's top edge so no gap shows between them.

The defect is the registration (`HW-13-0044`). `win.on(...)` runs in
`aboutToAppear`, the `@CustomDialog` content is rebuilt on **every** `open()`,
and there is no `off` anywhere in the project. Open the comment input five
times and five listeners are live; each one still calls `close()` on a
`CustomDialogController` belonging to a destroyed dialog. Storing the window
and the callback and releasing them in `aboutToDisappear` is the whole fix -
`off` needs the identical function reference, which is why the handler has to
be a field rather than an inline arrow.

**Submitting a comment — same file** (corrected, see `HW-13-0043`)

```typescript
TextInput({
  text: $$this.textInputText,
  placeholder: $r('app.string.dialogfirst_test'),
  controller: this.textController
})
  .onSubmit(() => {
    this.rightData.push(this.textInputText);
    this.textInputText = '';              // FIX: sample assigns $r('app.string.dialogfirst_test')
  })
  .defaultFocus(true)                     // raises the keyboard with the sheet
  .layoutWeight(1)
  .borderRadius(20);
```

**`$$this.textInputText` is a two-way binding**, so the field writes back into
the component's state without an `onChange`. That is what makes the reset
assignment meaningful - and what makes the shipped version wrong. The sample
resets to `$r('app.string.dialogfirst_test')`, which is the **same resource
used as the field's `placeholder`** (`HW-13-0043`). A placeholder is grey
hint text that is not content; assigning it to `text` makes it real content,
indistinguishable to the user from something they typed, and the next
`onSubmit` pushes the hint into the comment list as a comment.

Note also that `rightData` is declared `Array<string>` in `DialogFirst` and
`Array<ResourceStr>` in the consuming dialog, while `textInputText` is
`ResourceStr`. The list renders `Text(item)` so both work, but the two
declarations of one provided value should agree.

## Permissions & config

**None.** The sample declares no `requestPermissions`. The video is a bundled
media resource, comments are in-memory, and `display`, `window` and
`inputMethod` are all permission-free.

The configuration that matters is the router map wiring:

```json5
"pages": "$profile:main_pages",
"routerMap": "$profile:route_map",
```

`main_pages` lists only `pages/Index`; `route_map` lists `DialogFirst`. A
`NavDestination` reached by name must be in the router map and must **not** be
in `main_pages` - it is not a page, it is a destination inside one.

The ability also declares an `EntryBackupAbility` with
`$profile:backup_config`, the standard backup/restore extension; nothing in
this feature depends on it.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- The sheet starts a **second** `Video` on the same source rather than reusing
  the page's player, so the clip restarts from zero when the comments open and
  again when they close.
- Nothing resumes the video after the sheet pops: the `pushPathByName`
  callback builds a string and discards it. The page is left on a stopped
  player showing the paused-state play icon.
- Layout is pinned to a phone. `this.screenHeight - 87`, `top: 352`,
  `left: this.screenWidth - 59`, `bottom: 103` and the 144x253 thumbnail are
  absolute vp values derived from `display.getAllDisplays()`; on a tablet or a
  resized 2in1 window they are wrong, and `getAllDisplays` reports the display,
  not the window.
- `VideoData`, `TopTabContent`, `CommentDataSource` and the `@Observed Comment`
  class are defined but unused - the visible list is two hardcoded `@Builder`
  calls plus a `ForEach` over a plain `string[]`. The `IDataSource`
  infrastructure is there for a feed the sample never builds.
- Comments live in `@Provide rightData` for the life of the destination. There
  is no network, no persistence, no author, no timestamp on a sent comment.
- The like counts, the emoji button and the image button are static.
- The 400 ms `setTimeout` before `pop()` is duplicated from the transition's
  `duration`; changing one silently desynchronises the other.

## Pitfalls

- **`HW-13-0043`** (B/low, confirmed) — `onSubmit` resets the bound input to
  `$r('app.string.dialogfirst_test')`, the same resource the field uses as its
  `placeholder`. Under `$$` two-way binding that hint text becomes real
  content, so the next submit posts the placeholder as a comment. Fix: assign
  the empty string.
- **`HW-13-0044`** (B/low, confirmed) — `CommentDialog.aboutToAppear` calls
  `win.on('keyboardHeightChange', ...)` and the project contains no matching
  `off`. The dialog content is rebuilt on every `open()`, so listeners
  accumulate one per open and the stale ones keep firing `close()` on
  controllers whose dialogs are gone. Fix: store the window and the callback in
  fields and call `win.off(...)` in `aboutToDisappear`.

## References

- `documentation/harmonyos-references/02_application-framework/ts-basic-components-navigation.md` - `NavPathStack`, `pushPathByName`, `pop` and the pop callback
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-navigation
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-navdestination.md` - `NavDestinationMode.DIALOG`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-navdestination
- `documentation/harmonyos-guides/03_application-framework/arkts-navigation-navigation.md` - the router-map form of route registration
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-navigation-navigation
- `documentation/harmonyos-references/02_application-framework/ts-transition-animation-component.md` - `TransitionEffect.translate` and its appear/disappear semantics
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-transition-animation-component
- `documentation/harmonyos-references/02_application-framework/ts-methods-custom-dialog-box.md` - `CustomDialogController`, `customStyle`, `DialogAlignment`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-methods-custom-dialog-box
- `documentation/harmonyos-references/02_application-framework/js-apis-window.md` - `getLastWindow`, `keyboardHeightChange`, `off`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-window
- `documentation/harmonyos-references/02_application-framework/js-apis-inputmethod.md` - `getController().stopInputSession()`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-inputmethod
- `documentation/harmonyos-references/02_application-framework/js-apis-display.md` - `getAllDisplays` and why it is not the window size
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-display
- `MEDIA-15` - the same interaction rendered as a danmaku overlay instead of a sheet
