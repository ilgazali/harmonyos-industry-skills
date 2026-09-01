---
id: SOCIAL-22
title: History float bubble - a draggable edge-snapping button over Navigation that reopens visited web pages
industry: 14_social_communication
doc: huawei_industry_tree/14_social_communication/docs/22_chat_web_float.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/chat_web_float-0000002329723933
sample: huawei_industry_tree/14_social_communication/downloads/ChatWebFloat.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.ArkWeb", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.ImageKit", "@kit.PerformanceAnalysisKit"]
apis: [Stack, Navigation, NavPathStack, NavDestination, LocalStorage, "@LocalStorageLink", "@StorageProp", PanGesture, position, onAreaChange, RichEditor, RichEditorController, Web, webview, webPageSnapshot, cropSync, CustomDialogController, Marquee, onWillShow, hilog, image, window]
permissions: [ohos.permission.INTERNET]
min_api: 20
modules: [entry (entry)]
findings: [HW-14-0052, HW-14-0087, HW-14-0088]
status: verified-with-fixes
---

## When to use

Load this card when a user has to **leave a conversation to look at something
and then come back**, and the app must not make them find their way back
manually. The pattern: a small draggable bubble parked at the screen edge,
outside the navigation stack, holding a list of the pages visited from this
chat - each with a thumbnail taken from the live web view.

The bubble is a plain `Row` in a `Stack` that sits *above* `Navigation`, not a
system floating window. That is the important choice: it needs no window
permission, it lives and dies with the page, and it cannot be left behind on
the home screen if the app crashes. If you need the bubble to survive the app
going to background, you need a real `TYPE_FLOAT` window instead - see
`SOCIAL-21`, which tries that and does not finish it.

Beyond web history, the same three parts - a `Stack`-overlaid draggable widget,
a `LocalStorage`-backed list, and `NavDestination.onWillShow` to re-evaluate
visibility on every return - cover a minimised video call, a running download
tray, or a "continue where you left off" pill.

## Feature checklist

- A chat page with three link messages, an input row, and a scroll-to-bottom
  list.
- Tapping a link opens it in a `Web` destination inside the same navigation
  stack.
- The web page has a 更多 (more) entry that opens a bottom dialog with a
  "float window" action.
- Choosing it snapshots the rendered page, crops the snapshot to a 180x120
  card ratio, and pops back to the chat carrying URL, title and thumbnail.
- A bubble appears at the right edge of the chat page once at least one page
  has been saved.
- Dragging the bubble moves it, and releasing it snaps it to whichever
  horizontal edge is nearer; it cannot be dragged off screen.
- Tapping the bubble opens a two-column grid of saved pages; tapping a card
  reopens that URL, and each card has a delete badge.
- Clearing the list, or deleting the last entry, hides the bubble again.

## Architecture

One `entry` module. `HomePage` is the only `@Entry`; everything the user sees
is a `NavDestination` inside it.

```
entry/src/main/ets
├── common/Constants.ets            FLOAT_BUTTON_SIZE = 40, the one shared number
├── component
│   ├── Chat.ets                    the chat NavDestination: list, links, float refresh
│   ├── ChatFloatPage.ets           the saved-pages grid, with clear and per-card delete
│   ├── CustomRichEditor.ets        the input row; owns a module-level RichEditorController
│   └── WebPage.ets                 the Web NavDestination + the bottom CustomDialog
├── entryability/EntryAbility.ets   full-screen layout, avoid areas -> AppStorage
├── entrybackupability/
├── interface/WebInfo.ets           { url, title, snapshot: image.PixelMap | undefined }
├── pages/HomePage.ets              @Entry: Stack(Navigation + the bubble), drag and snap
└── utils/Logger.ets
```

The documented tree matches the zip exactly.

**The design decision worth copying** is that the bubble lives in `HomePage`,
one level *outside* the `Navigation` it overlays, while its visibility is owned
by the destinations underneath it. `HomePage` renders:

```typescript
Stack() {
  Navigation(this.navPathStack) { }
    .navDestination(this.navDestinations);
  Row() { Image($r('app.media.float_button')) }   // the bubble
    .position({ x: this.x, y: this.y })
}
```

Because the `Stack` is the page root, the bubble draws over every destination
and survives every push and pop - no destination has to re-render it. But
`Chat` is the one that decides whether it should be seen, from its own
lifecycle:

```typescript
.onWillShow(() => {
  this.refreshFloatVisible();      // showFloat = webInfos.length > 0
})
```

That split is the point. Position is a property of the page; relevance is a
property of the screen you are on. `onWillShow` fires on every return to the
chat - after a pop from the web page, after a pop from the history grid - so
one callback covers all the ways the list can have changed.

## Implementation steps

1. **Make the page a `Stack`** whose first child is `Navigation` sized
   `100%/100%` and whose second is the bubble. Push the first destination from
   `aboutToAppear` so the chat is what the user sees.
2. **Measure the container with `onAreaChange`,** not with constants: store
   `containerWidth`/`containerHeight`, and seed the bubble at the right edge,
   vertically centred. Compare `oldValue`/`newValue` per axis so a rotation or
   a window resize re-seeds only the axis that moved.
3. **Move the bubble in `PanGesture.onActionUpdate`** by adding
   `event.offsetX/Y` to a remembered origin, clamping to
   `[0, container - FLOAT_BUTTON_SIZE]` on both axes.
4. **Snap in `onActionEnd`**: if the centre is past half the width, go to the
   right edge, otherwise to the left, then commit the result into
   `originX/originY` - the offsets in the next gesture are relative to that
   origin, so forgetting the commit makes the bubble jump.
5. **Keep the saved pages in one `LocalStorage` list** exposed as
   `@LocalStorageLink('webInfos')` in every component that touches it. Do not
   also construct a private `new LocalStorage()` (`HW-14-0052`).
6. **Recompute visibility from `NavDestination.onWillShow`,** not from the push
   site, so deletion in the grid and clearing both take effect.
7. **Return the visited page through `pop(result)` and `onPop`**: the web page
   pops a `WebInfo`, the chat's `onPop` callback de-duplicates by URL and
   appends.
8. **Take the thumbnail with `webPageSnapshot`** on the component's `id`, crop
   it to the card aspect ratio with `cropSync`, and only then pop - the
   snapshot is asynchronous and the page must still be alive when it is taken.

## Verified snippets

All snippets are from `ChatWebFloat.zip`. Corrected forms are marked.

**Drag, clamp and snap - `entry/src/main/ets/pages/HomePage.ets`** (as shipped)

```typescript
import { FLOAT_BUTTON_SIZE } from '../common/Constants';

@State x: number = 0;
@State y: number = 0;
originX: number = 0;
originY: number = 0;
@State containerWidth: number = 0;
@State containerHeight: number = 0;

// ... inside the Stack, over the Navigation:
Row() {
  Image($r('app.media.float_button')).size({ width: 25, height: 25 });
}
.size({ width: FLOAT_BUTTON_SIZE, height: FLOAT_BUTTON_SIZE })
.position({ x: this.x, y: this.y })
.shadow({ radius: 10, color: Color.Gray })
.gesture(
  PanGesture()
    .onActionEnd(() => {
      if (this.x < this.containerWidth / 2) {
        this.x = 0;
      } else {
        this.x = this.containerWidth - FLOAT_BUTTON_SIZE;
      }
      this.originX = this.x;              // commit, or the next drag starts from the old origin
      this.originY = this.y;
    })
    .onActionUpdate((event: GestureEvent) => {
      this.x = (this.originX + event.offsetX < 0 ? 0 : this.originX + event.offsetX);
      this.y = this.originY + event.offsetY < 0 ? 0 : this.originY + event.offsetY;
      this.x = this.x + FLOAT_BUTTON_SIZE > this.containerWidth ?
        this.containerWidth - FLOAT_BUTTON_SIZE : this.x;
      this.y = this.y + FLOAT_BUTTON_SIZE > this.containerHeight ?
        this.containerHeight - FLOAT_BUTTON_SIZE : this.y;
    })
)
.visibility(this.showFloat ? Visibility.Visible : Visibility.Hidden);
```

**`offsetX` is cumulative within one gesture, not incremental.** That is why
the handler adds it to a remembered `originX` instead of to the current `x` -
adding to `x` would accelerate the bubble quadratically. The pair
`origin`/`current` is the whole state machine: `onActionUpdate` derives,
`onActionEnd` commits.

Clamping happens twice per axis for a reason: the first expression stops the
bubble at 0, the second at `container - size`. Both use the bubble's own size,
which is why `FLOAT_BUTTON_SIZE` is a shared constant and not a literal in the
layout - the geometry and the clamp must never disagree.

Note `.visibility(... : Visibility.Hidden)` rather than an `if`: `Hidden`
retains the layout node, so `x`/`y` survive while the bubble is invisible and
it reappears exactly where the user left it.

**The storage that should not be there - same file** (corrected, see `HW-14-0052`)

```typescript
@Entry
@Component
struct HomePage {
  @LocalStorageLink('showFloat') showFloat: boolean = false;
  // FIX: the sample also declares `localStorage: LocalStorage = new LocalStorage();`
  //      and writes the two seed values into it - a store nothing ever reads.

  aboutToAppear(): void {
    let webInfos: WebInfo[] = [];
    this.localStorage.setOrCreate('webInfos', webInfos);     // writes into the detached instance
    this.localStorage.setOrCreate('showFloat', false);
    this.navPathStack.pushPath({ name: 'Chat' });
  }
}
```

**`@LocalStorageLink` binds to the storage the page was given, not to any
instance you happen to hold.** `HomePage` is declared as a bare `@Entry`, so
its links resolve against the implicit page-level `LocalStorage`; the
`new LocalStorage()` field is a second, unrelated store. The two
`setOrCreate` calls in `aboutToAppear` therefore write values that no
`@LocalStorageLink` in this app can see, and the feature only works because
`[]` and `false` happen to be the declared defaults of the links themselves.

Two ways out, both one line. Either bind it -
`let storage = new LocalStorage(); @Entry(storage)` - and keep the seeding, or
delete the field and the two writes and let the defaults stand. What you must
not keep is the current arrangement, where a future non-default seed value (say
a restored history list) would silently never reach the UI.

**Snapshot, crop, and pop the result - `entry/src/main/ets/component/WebPage.ets`** (as shipped)

```typescript
const WEB_PAGE_ID = 'webPage';

popReturnWebInfo() {
  this.controller.webPageSnapshot({ id: WEB_PAGE_ID, size: { width: '100%', height: '100%' } },
    (error, result) => {
      if (error) {
        Logger.error(`ErrorCode: ${(error as BusinessError).code},  Message: ${(error as BusinessError).message}`);
        return;
      }
      if (result) {
        if (!this.webInfo.snapshot && result.imagePixelMap) {
          let imageInfo = result.imagePixelMap.getImageInfoSync();
          let width = imageInfo.size.width;
          let height = width * 120 / 180;                  // the grid card's aspect ratio
          result.imagePixelMap.cropSync({ size: { width: width, height: height }, x: 0, y: 0 });
          this.webInfo.snapshot = result.imagePixelMap;
        }
        this.navPathStack.pop(this.webInfo);
      }
    });
}
```

**The crop is what makes the grid look like a set of cards rather than a set of
screenshots.** `webPageSnapshot` returns the full rendered page; cropping to
`width x width*120/180` from the origin keeps the top of the page - the part
that identifies it - at exactly the aspect ratio the `GridItem` renders at
180x120. Doing it here rather than with `objectFit` in the grid means the
oversized `PixelMap` is discarded immediately instead of being held for every
saved entry.

`pop(this.webInfo)` is inside the callback, not after it. The snapshot needs
the web component alive, and popping first would tear it down; this ordering is
the reason the "more" dialog closes before the pop rather than as part of it.

**Receiving the result back in the chat - `entry/src/main/ets/component/Chat.ets`** (as shipped)

```typescript
Span(content)
  .decoration({ type: TextDecorationType.Underline })
  .onClick(() => {
    this.showFloat = false;
    this.navPathStack.pushPath({
      name: 'WebPage', param: content, onPop: (popInfo) => {
        let webInfo = popInfo.result as WebInfo;
        if (webInfo) {
          let found = this.webInfos.filter((value) => {
            return value.url === webInfo.url;
          });
          if (found.length === 0) {
            this.webInfos.push(webInfo);
          }
        }
      }
    });
  })
  .fontColor($r('app.color.clear_font_color'));
```

**`onPop` on the push is the return channel.** The chat states, at the moment
it navigates away, what to do with whatever comes back - so there is no shared
"pending result" field and no ordering hazard between the pop and the
re-render. The de-duplication by `url` is what keeps the grid from filling with
the same page; note that it also means a page whose title or snapshot changed
is not refreshed.

`this.showFloat = false` before the push hides the bubble over the web page.
It is set back to `webInfos.length > 0` by `onWillShow` on the way back, which
is the same call that covers deletions made in the grid.

The link detection is a module-level `isUrl(str)` regular expression applied
per `Span` inside one `Text`, so a message can mix prose and links on one
baseline.

## Permissions & config

```json5
"requestPermissions": [
  { "name": "ohos.permission.INTERNET" }
]
```

- `INTERNET` is `system_grant`; nothing is prompted, and no `reason` is needed.
- No window permission is required, because the bubble is an in-page component,
  not a system floating window. This is the main practical advantage of the
  approach.
- `Web` is configured defensively on the destination: `fileAccess(false)` and
  `geolocationAccess(false)`, with `javaScriptAccess` left on.
- Routing is by `navDestination` builder inside `Navigation`, not by a route
  map - all three destination names (`Chat`, `WebPage`, `ChatFloatPage`) are
  resolved in one `@Builder` `if/else` chain in `HomePage`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`.
- The bubble is in-page only. Backgrounding the app takes it with the app; it
  is not a `window.WindowType.TYPE_FLOAT` overlay.
- Nothing is persisted. `webInfos` lives in page storage, so relaunching loses
  the history and the thumbnails.
- Snapshots are held as live `image.PixelMap` objects in the list with no cap
  and no `release()`, so a long browsing session grows memory monotonically.
- `ChatFloatPage`'s grid `ForEach` has no key generator, so ArkUI falls back to
  index-based keys - acceptable for a small list, but deleting from the middle
  re-renders every card after it.
- The list `ForEach` in `Chat` keys on `JSON.stringify(item)_index`. Appending
  the index is what saves it; this sample is not one of the six instances of
  the industry-wide broken-key defect recorded on `SOCIAL-08`, but it is one
  edit away from being one.
- The chat messages are three string-resource URLs; there is no back end and
  the send button only appends to a local array.

## Pitfalls

- **`HW-14-0052`** (B/low, confirmed): `HomePage` constructs a private
  `new LocalStorage()` and seeds `webInfos`/`showFloat` into it, but the page
  is a bare `@Entry`, so every `@LocalStorageLink` in the app binds the
  implicit page storage instead. The writes go to a store nothing reads and the
  feature works only by virtue of the decorators' own defaults; any non-default
  seed value would silently never appear. Fix: bind the instance with
  `@Entry(storage)` or drop the field and the two `setOrCreate` calls.
- **Not a numbered finding, but adjacent:** `WebPage.aboutToAppear` reads its
  URL with `getParamByName('WebPage')[0]`, which returns the parameter of the
  *first* `WebPage` entry in the stack. `ChatFloatPage` uses `replacePath`, so
  the stack never holds two - but the moment a second web destination can be
  pushed, this reads the wrong URL. Prefer `NavDestination.onReady` and
  `context.pathInfo.param`, as `SOCIAL-24` does.

## References

- `documentation/harmonyos-references/02_application-framework/js-service-widget-container-stack.md` - `Stack` and overlay ordering
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-service-widget-container-stack
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-navigation.md` - `Navigation`, `NavPathStack`, `navDestination`, `onPop`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-navigation
- `documentation/harmonyos-guides/03_application-framework/arkts-navigation-navigation.md` - destination lifecycle, `onWillShow`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-navigation-navigation
- `documentation/harmonyos-guides/03_application-framework/arkts-localstorage.md` - `LocalStorage`, `@LocalStorageLink`, `@Entry(storage)`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-localstorage
- `documentation/harmonyos-references/02_application-framework/ts-basic-gestures-pangesture.md` - `PanGesture`, `offsetX`/`offsetY` semantics
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-gestures-pangesture
- `documentation/harmonyos-references/02_application-framework/arkts-apis-webview-webviewcontroller.md` - `webPageSnapshot`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-webview-webviewcontroller
- `documentation/harmonyos-references/04_media/arkts-apis-image-pixelmap.md` - `cropSync`, `getImageInfoSync`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-image-pixelmap
- `documentation/harmonyos-guides/04_system/permissions-for-all.md` - `ohos.permission.INTERNET`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all
- `SOCIAL-24` - the same chat-to-Web destination hop, reached by decoding a QR code
- `SOCIAL-21` - the system floating window this sample deliberately avoids
