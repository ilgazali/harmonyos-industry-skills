---
id: UTIL-14
title: Rate this app - three ways to reach the AppGallery review page, and which one to wire
industry: 15_utilities
doc: huawei_industry_tree/15_utilities/docs/14_comment.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/comment-0000002292394860
sample: huawei_industry_tree/15_utilities/downloads/comment.zip
kits: ["@kit.AbilityKit", "@kit.AppGalleryKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: [commentManager, "commentManager.showCommentDialog", "UIAbilityContext.startAbility", "UIAbilityContext.openLink", Want, hilog, window, "@StorageProp"]
permissions: []
min_api: 20
modules: [entry (entry)]
findings: [HW-15-0036, HW-15-0101]
status: verified-with-fixes
---

## When to use

Load this card when you need a **"rate us" entry point** - a row in the
settings page, a prompt after a successful order, a five-star item in a
profile menu - and you want it to land on the AppGallery review sheet rather
than on a home-grown feedback form.

There are three routes, and they are not interchangeable. Deep Link
(`startAbility` with a `store://` URI) and App Linking (`openLink` with an
`https://` URL) both open the **AppGallery detail page of a named bundle** and
ask it to continue to the write-review page; they can target any app, including
someone else's. The comment service (`commentManager.showCommentDialog`) opens
an **in-app rating dialog for the current app only**, and the system rate-limits
it: after one use, it will not reappear until a new version has shipped and a
year has passed.

Choose by what you are pointing at. A menu item that says "rate this app"
wants `showCommentDialog` - it keeps the user inside your app and costs one
call. A link to a *different* product (a companion app, a store listing you
promote) wants App Linking. Deep Link is the fallback when App Linking is not
configured for the target. **The sample ships only the App Linking route wired
to a button, and points it at a third-party bundle** (`HW-15-0036`) - read the
pitfalls before treating the demo as a template.

## Feature checklist

- A settings-style page: an avatar card with a name and a masked phone number,
  then a list of five rows, then a footer and a sign-out image.
- The 五星好评 (five-star review) row is tappable and opens the AppGallery
  write-review flow for a given bundle.
- Success and failure are logged through `hilog` with the error code and
  message; nothing is shown to the user.
- The page is laid out full screen, inset by the status bar height at the top
  and the navigation indicator height at the bottom.
- The other four rows (今日签到, 签到排名, 服务协议, 隐私政策) are inert.

## Architecture

One `entry` module, one page. There is no model layer, no view layer and no
state - the whole sample is a static settings screen plus three methods.

```
entry/src/main/ets
├── entryability/EntryAbility.ets
├── entrybackupability/EntryBackupAbility.ets
└── pages/JumpToComment.ets     struct StartAppGalleryDetailAbilityView, 303 lines:
                                the three launch methods (30-65) then one long build()
```

The documented tree matches the zip. Note the file name and the struct name
disagree - the file is `JumpToComment.ets`, the component is
`StartAppGalleryDetailAbilityView`, and `TAG` repeats the struct name.

**The design decision worth copying** is the context capture:

```typescript
private context: common.UIAbilityContext = this.getUIContext().getHostContext() as common.UIAbilityContext;
```

All three routes need a `UIAbilityContext` - `startAbility` and `openLink` are
its methods, and `showCommentDialog` takes it as its only argument. Resolving
it once as a private field, rather than per call site, is what keeps the three
methods one-liners. The `as` cast is unavoidable: `getHostContext()` is typed
`Context | undefined`, and the ability-level methods live on the narrower type.

**The structural choice worth avoiding** is everything else. The five list rows
are 60 lines of copy-pasted `Row`/`Image`/`Text` with hardcoded `'12%'` /
`'85%'` widths and per-row `Image($r('app.media.line'))` separators, and only
one of them carries an `onClick`. A `ForEach` over a small array of
`{ icon, label, action }` would have been a third of the size and would have
made the dead rows obvious. Copy the three methods, not the page.

## Implementation steps

1. **Resolve the ability context once** as a component field, cast to
   `common.UIAbilityContext`.
2. **Pick one route per entry point.** Do not ship all three behind one
   button; they lead to visibly different places (`HW-15-0036`).
3. **For the in-app dialog**, call `commentManager.showCommentDialog(context)`.
   It always targets the *current* app - there is no bundle parameter.
4. **Wrap `showCommentDialog` in `try/catch` as well as `.catch`.** It can
   throw synchronously on a parameter or capability error, and reject
   asynchronously; the sample handles both, and that is correct.
5. **For App Linking**, build
   `https://appgallery.huawei.com/app/detail?id=<bundle>&action=write-review`
   and call `openLink(link, { appLinkingOnly: false })`. `false` means "fall
   back to a browser/deep-link resolution if App Linking is not available";
   `true` fails instead, which is what you want only if a wrong landing page
   is worse than no landing page.
6. **For Deep Link**, build a `Want` with
   `action: 'ohos.want.action.appdetail'` and the `store://` URI, then
   `startAbility(want)`. The `action=write-review` query parameter is what
   makes the detail page continue to the review sheet.
7. **Surface failures to the user.** All three paths can fail on a device with
   no AppGallery; the sample only logs (`HW-15-0036` context). A toast on the
   `.catch` is the minimum.
8. **Point the bundle at yourself.** The sample passes
   `'com.huawei.hmos.vmall'`, a different product; in your app the argument
   should be your own bundle name, or `showCommentDialog` with no argument at
   all.

## Verified snippets

All snippets are from `comment.zip`, `entry/src/main/ets/pages/JumpToComment.ets`.

**Route 1 — Deep Link** (as shipped; unreachable in the sample, see `HW-15-0036`)

```typescript
// 通过DeepLink拉起应用市场指定的应用写评论页
startDetailWithDeepLink(bundleName: string): void {
  let want: Want = {
    // 隐式指定action为ohos.want.action.appdetail
    action: 'ohos.want.action.appdetail',
    // bundleName为需要拉起写评论页的应用包名，action隐式指定为write-review
    uri: `store://appgallery.huawei.com/app/detail?id=${bundleName}&action=write-review`
  };
  this.context.startAbility(want).then(() => {
    hilog.info(0x0001, TAG, `Succeeded in starting Ability successfully.`);
  }).catch((error: BusinessError) => {
    hilog.error(0x0001, TAG, `Failed to startAbility. Code: ${error.code}, message is ${error.message}`);
  });
}
```

**The `Want` carries no `bundleName` field, and that is deliberate.** This is
an *implicit* want: the action `ohos.want.action.appdetail` is what selects the
AppGallery, and the target application is named inside the `store://` URI as
the `id` query parameter. Setting `want.bundleName` to your own package here -
an easy mistake - would make the want explicit and route it back into your own
app.

Two parameters in the URI do the work: `id` picks which app's detail page
opens, and `action=write-review` tells that page to continue straight to the
review composer instead of stopping at the listing.

**Route 2 — App Linking** (as shipped; the only route wired to a tap)

```typescript
// 通过AppLinking拉起应用市场指定的应用写评论页
startDetailWithAppLinking(bundleName: string): void {
  let link: string = `https://appgallery.huawei.com/app/detail?id=${bundleName}&action=write-review`;
  this.context.openLink(link, { appLinkingOnly: false }).then(() => {
    hilog.info(0x0001, TAG, `Succeeded in starting AppLinking successfully.`);
  }).catch((error: BusinessError) => {
    hilog.error(0x0001, TAG, `Failed to start AppLinking. Code: ${error.code}, message is ${error.message}`);
  });
}
```

**Same destination, different transport.** The query string is identical to the
Deep Link one; only the scheme and the API change. App Linking is the
preferred route because the `https://` URL is verified against the domain's
association file, so it cannot be hijacked by another app claiming the scheme -
which is exactly the risk a `store://` URI carries.

`appLinkingOnly: false` is the permissive setting: if App Linking cannot
resolve the URL on this device, the system is allowed to fall back to ordinary
URL handling, which usually means a browser landing on the same page. Set it to
`true` when landing in a browser would be worse than failing outright.

**Route 3 — the in-app rating dialog** (as shipped; unreachable, see `HW-15-0036`)

```typescript
import { commentManager } from '@kit.AppGalleryKit';

// 通过应用内评论弹窗写评论，弹窗指向为当前应用。
// 待新版本发布且距上次评论已经一年，可继续弹出评分弹窗。
startCommentDialog() {
  try {
    commentManager.showCommentDialog(this.context).then(() => {
      hilog.info(0, 'TAG', "succeeded in showing commentDialog.");
    }).catch((error: BusinessError<Object>) => {
      hilog.error(0, 'TAG', `showCommentDialog failed, Code: ${error.code}, message: ${error.message}`);
    });
  } catch (error) {
    hilog.error(0, 'TAG', `showCommentDialog failed, Code: ${error.code}, message: ${error.message}`);
  }
}
```

**This is the route most apps actually want, and the one the sample leaves
dead.** It takes no bundle name because it can only ever rate the calling
application, and it does not leave the app - the sheet is drawn over your page.
The comment above it states the frequency rule that the API enforces for you:
once a user has been shown the dialog, it will not appear again until a new
version has been published *and* a year has passed since the last review. Your
code cannot detect that state, so the resolved promise does not mean a dialog
was seen - never chain a "thanks for rating" message to it.

The double error handling is correct and worth copying: the synchronous
`try/catch` catches parameter and capability errors thrown before the promise
exists, and the `.catch` handles the asynchronous failure. Note the `hilog`
calls here pass the literal string `'TAG'` and domain `0`, unlike the other two
methods which use the `TAG` constant and domain `0x0001` - a small
inconsistency that makes this method harder to grep for in a log.

**The single wired tap — same file** (as shipped)

```typescript
Row() {
  Row() {
    Image($r('app.media.path2')).width(24).height(24);
  }
  .width('12%')
  .height(48);

  Row() {
    Text($r('app.string.wuxinghaoping'))     // 五星好评 — "five-star review"
      .fontSize(14)
      .fontWeight(FontWeight.Medium);
    Image($r('app.media.advanceIcon')).width(6.74).height(12.81);
  }
  .width('85%')
  .height(48)
  .justifyContent(FlexAlign.SpaceBetween);
}
.padding(20)
.width('100%')
.height(48)
.onClick(() => {
  this.startDetailWithAppLinking('com.huawei.hmos.vmall');
});
```

**This is the whole of `HW-15-0036` in one handler.** The row labelled 五星好评
("five-star review") is the only interactive element on the page, it calls one
of the three documented methods, and the bundle it passes -
`com.huawei.hmos.vmall`, Huawei's Vmall shopping app - is not the sample. A
user tapping "rate this app" is taken to a different product's review page.

For your own adoption the handler should read
`this.startCommentDialog()` (rate this app, in place) or
`this.startDetailWithAppLinking(<your own bundle name>)`. The `.onClick` also
belongs on the outer `Row` as it is here rather than on the `Text`, so the
whole 48 vp strip is the tap target.

## Permissions & config

**None.** The sample declares no `requestPermissions`. Launching the AppGallery
by implicit want or by App Linking needs no manifest entry on the caller's
side.

`module.json5` restricts `deviceTypes` to `["phone"]` only - narrower than the
other samples in this industry, and appropriate: the AppGallery review flow is
a phone-shaped experience.

`@kit.AppGalleryKit` is imported directly with no `dependencies` entry, so
`commentManager` resolves from the SDK rather than from an OHPM package.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **All three routes require the AppGallery to be present.** On a device
  without it, `startAbility` and `openLink` reject and `showCommentDialog`
  fails; there is no offline fallback.
- **The rating dialog cannot be shown on demand.** After one appearance it is
  suppressed until a new version has been released and a year has elapsed. A
  "rate us" button using it will silently do nothing for most users, which is
  an argument for the App Linking route on a permanently visible menu row.
- The resolved promise means "the launch/dialog request succeeded", not "the
  user reviewed the app". There is no callback carrying the rating.
- App Linking requires the target domain association to be configured for the
  bundle being pointed at; with `appLinkingOnly: false` an unconfigured target
  degrades to a browser rather than failing.
- The page is a mock: the avatar, the name 张小旭, the masked number
  `189****1234` and the four other rows are static.

## Pitfalls

- **`HW-15-0036` — the document promises three comment flows; two are dead
  code.** (E/low, confirmed) `startDetailWithDeepLink` and `startCommentDialog`
  are defined at `JumpToComment.ets:30-65` and never called; the page's single
  `onClick` (`:197-199`) invokes only `startDetailWithAppLinking`. A reader
  running the sample can exercise one third of what the page documents.
  **Fix:** wire the two dead methods to their own rows, or reduce the document
  to the flow that ships.
- **The one wired route targets someone else's bundle.**
  `startDetailWithAppLinking('com.huawei.hmos.vmall')` opens the review page
  for Huawei Vmall, not for this app. Substitute your own bundle name, or use
  `showCommentDialog`, which cannot be pointed at the wrong app.
- **Failures are logged, never surfaced.** Every `.catch` calls `hilog.error`
  and stops, so on a device without the AppGallery the row appears to do
  nothing at all. Add a toast.
- **`hilog` usage is inconsistent between the three methods** - domain
  `0x0001` with the `TAG` constant in two of them, domain `0` with the literal
  string `'TAG'` in the third. Trivial, but it defeats log filtering exactly
  where the least-tested path is.
- **Four of the five list rows have no handler.** If you lift this page as a
  settings screen, they are decoration, not TODOs that the framework will
  flag.

## References

- `documentation/harmonyos-guides/07_application-services/appgallery-comment.md` - the three flows, the Deep Link and App Linking URL formats, and the frequency rule
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/appgallery-comment
- `documentation/harmonyos-references/06_application-services/appgallery-commentmanager.md` - `commentManager.showCommentDialog` and its error codes
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/appgallery-commentmanager
- `documentation/harmonyos-references/02_application-framework/js-apis-inner-application-uiabilitycontext.md` - `startAbility` and `openLink`, including `appLinkingOnly`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-inner-application-uiabilitycontext
- `documentation/harmonyos-references/02_application-framework/js-apis-app-ability-want.md` - implicit wants and the `action` / `uri` fields
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-app-ability-want
- `huawei_industry_tree/15_utilities/docs/14_comment.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/comment-0000002292394860
