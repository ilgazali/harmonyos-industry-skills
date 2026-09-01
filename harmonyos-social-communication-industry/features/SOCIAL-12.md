---
id: SOCIAL-12
title: Chat link interception - onLoadIntercept whitelists a navigation and routes everything else to a local block page
industry: 14_social_communication
doc: huawei_industry_tree/14_social_communication/docs/12_h5_interception.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/h5_interception-0000002282125390
sample: huawei_industry_tree/14_social_communication/downloads/聊天页-网页访问拦截示例代码.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.ArkWeb", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit"]
apis: [hilog, webview, window]
permissions: [ohos.permission.INTERNET]
min_api: 20
modules: [entry]
findings: [HW-14-0026, HW-14-0027, HW-14-0087, HW-14-0088]
status: verified-with-fixes
---

## When to use

Load this card when **a link inside user-generated content opens a web view
inside your app** and you are responsible for where it goes. A chat message, a
forum post, a comment, a push notification landing page - all of them can carry
a URL the app did not author, and an in-app `Web` component will follow it, plus
every redirect it hits afterwards.

The mechanism is `Web.onLoadIntercept`: it fires before each navigation, hands
you the request, and your return value decides - `false` lets it proceed,
`true` cancels it. The sample uses the cancel branch to load a bundled
`blocked.html` from `rawfile`, so the user sees an explanation instead of a
blank page.

The card also covers the two halves that make link interception usable in
practice: **turning plain text into tappable link spans** in the message
composer (a regex splits the typed text into alternating text and URL runs, and
each run becomes a `Span` or a link-coloured `Span`), and **the whitelist
itself**. Read `HW-14-0026` first: the shipped whitelist is a substring test,
and a substring test on a URL is not a whitelist.

## Feature checklist

- A chat page seeded with one received message containing a link and one plain
  reply.
- Typing a message containing a URL and sending it splits the text so the URL
  renders in link blue and the rest renders as normal text.
- Tapping a link span pushes a web page destination inside the same
  `Navigation`.
- The web page shows a title bar with the page title and the current URL, and a
  close button that pops the destination.
- A navigation to a whitelisted host loads normally and the address in the title
  bar updates.
- A navigation to any other host is cancelled and the local block page is shown
  instead.
- Loading a local `resource://rawfile/` page hides the title/URL bar.
- The system back gesture walks the web history first, and only pops the
  destination once the history is exhausted.

## Architecture

One `entry` module, two pages, one component, one model, two utilities.

```
entry/src/main/ets
├── components/CustomRichEditor.ets   the input row: URL splitting, send, keyboard height
├── entryability/EntryAbility.ets     full screen, avoid areas -> AppStorage (raw px)
├── model/MessageContent.ets          MessageType (TEXT/LINK/IMAGE), MsgContent, the seed MSG
├── pages/ChatPage.ets                @Entry: Navigation host, the bubble list
├── pages/WebPage.ets                 the NavDestination holding the Web component
├── utils/Logger.ets                  hilog wrapper
└── utils/UrlUtils.ets                parseUrl / isUrl / canUrlAccess
entry/src/main/resources/rawfile
└── blocked.html                      the local interception page
```

The documented 工程目录 matches the zip. (The industry tree-mismatch
systematic `HW-14-0001` covers four other social samples, not this one.)

**The design decision worth copying** is that a message is
`MsgTextImage[]` - a list of typed runs - rather than a string with markup.
`MessageType` is `TEXT | LINK | IMAGE`, the composer produces the runs, and
`ChatPage` renders each run as a `Span`, a link-coloured `Span` with an
`onClick`, or an `ImageSpan`, all inside a single `Text`. Because they are
`Span`s in one `Text`, the link wraps and flows with the surrounding sentence
instead of being a separate component with its own line box - which is what a
naive "component per link" approach gets wrong. Classification happens **once,
at send time**, so rendering never re-parses and the link boundaries can never
disagree between two renders.

The interception itself lives in exactly one place: `UrlUtils.canUrlAccess`,
called from `WebPage.onLoadIntercept`. That is the right shape - one predicate,
one call site, easy to test. It is also why fixing `HW-14-0026` is a
single-function change.

## Implementation steps

1. **Model a message as typed runs**, not as a string: `MessageType.TEXT`,
   `LINK`, `IMAGE`.
2. **Split the composed text with one regex, applied twice** - `match` for the
   URLs, `split` for the gaps - and interleave the two results so run order is
   preserved.
3. **Render the runs as `Span`s inside one `Text`**, giving the LINK runs a
   colour and an `onClick` that pushes the web destination by name with the URL
   as the parameter.
4. **Declare the web page in the router map**, so the chat page pushes by name
   and never imports it.
5. **Read the URL in `NavDestination.onReady`** from `context.pathInfo.param`
   and bind it to `Web({ src: this.url })`.
6. **Decide in `onLoadIntercept`**: `false` proceeds, `true` cancels. On cancel,
   `controller.loadUrl($rawfile('blocked.html'))`.
7. **Parse the URL and compare the host** - do not search the whole URL for a
   substring (`HW-14-0026`).
8. **Update the displayed address only for remote pages**, so the local block
   page does not put `resource://rawfile/blocked.html` in the title bar.
9. **Handle back by walking the web history first**: `accessBackward()` then
   `backward()` and return `true`; otherwise return `false` and let the
   destination pop.
10. **Unregister the `keyboardHeightChange` listener in `aboutToDisappear`**
    (`HW-14-0027`).

## Verified snippets

All snippets are from `聊天页-网页访问拦截示例代码.zip` (the project directory is
`H5Interception`). Corrected forms are marked.

**The interception point — `entry/src/main/ets/pages/WebPage.ets`** (as shipped)

```typescript
import { webview } from '@kit.ArkWeb';
import { UrlUtils } from '../utils/UrlUtils';

private controller: WebviewController = new webview.WebviewController();
@State title: string = '';
@State url: string = '';
@State canShowTitle: boolean = true;

Web({
  src: this.url,
  controller: this.controller
})
  .layoutWeight(1)
  .zoomAccess(false)
  .javaScriptAccess(true)
  .onLoadIntercept((event) => {
    let url = event.data.getRequestUrl();
    if (UrlUtils.canUrlAccess(url)) {
      if (!url.startsWith('resource://')) {
        this.url = url;                  // remote page: show the address
        this.canShowTitle = true;
      } else {
        this.canShowTitle = false;       // local block page: hide the bar
      }
      return false;                      // false = allow
    } else {
      this.controller.loadUrl($rawfile('blocked.html'));
      return true;                       // true = cancel this navigation
    }
  })
  .onPageBegin(() => {
    this.title = this.controller.getTitle();
  })
  .onTitleReceive((event) => {
    this.title = event.title;
  })
```

**The return value is the whole API, and it reads backwards.** `true` means
"I have handled this, do not load it"; `false` means "carry on". Getting it the
wrong way round produces an app that blocks exactly the sites it meant to
allow, so it is worth a comment at the call site.

`onLoadIntercept` fires for **every** navigation the web engine starts, not
only for the URL you passed in `src` - redirects, `window.location` assignments
and link clicks inside the loaded page all come through here. That is what makes
it the right hook: a check performed once before `src` is set would be defeated
by the first 302.

Note the `resource://` branch. `loadUrl($rawfile(...))` itself triggers an
interception pass, so the block page must be allowed through; the sample
handles it in the predicate and uses the same test to hide the address bar,
since showing `resource://rawfile/blocked.html` to a user would be noise.
`javaScriptAccess(true)` is enabled here - reasonable for a browser view, and
the reason a correct whitelist matters rather than a cosmetic one.

**The whitelist — `entry/src/main/ets/utils/UrlUtils.ets`** (corrected, see `HW-14-0026`)

```typescript
import { url as urlKit } from '@kit.ArkTS';
import { LOGGER } from './Logger';

export class UrlUtils {
  private static readonly ALLOWED_HOSTS: string[] = ['huawei.com'];

  public static canUrlAccess(target: string): boolean {
    // FIX: the sample is `target.indexOf('resource://rawfile/') !== -1`
    if (target.startsWith('resource://rawfile/')) {
      return true;
    }
    // the seeded link is scheme-less ('www.huawei.com'), so normalise before parsing
    const normalized: string = /^[a-z][a-z0-9+.\-]*:\/\//i.test(target) ? target : `https://${target}`;
    try {
      const host: string = urlKit.URL.parseURL(normalized).hostname.toLowerCase();
      for (const allowed of UrlUtils.ALLOWED_HOSTS) {
        // FIX: the sample is `target.indexOf('www.huawei') !== -1` - anywhere in the URL
        if (host === allowed || host.endsWith(`.${allowed}`)) {
          return true;
        }
      }
    } catch (e) {
      LOGGER.info(`Unparsable URL blocked: ${target}`);
      return false;
    }
    LOGGER.info(`Address Access Blocked: ${target}`);
    return false;
  }
}
```

**`indexOf` on a URL asks the wrong question.** The token `www.huawei` can
appear in the path, the query, the fragment, the userinfo or the subdomain of a
host an attacker controls, and every one of those passes the shipped test:
`https://evil.com/www.huawei`, `https://evil.com/?r=www.huawei`,
`https://www.huawei.evil.com/`. The sample's entire reason to exist is the
document's claim 用户只能访问白名单内的网址 (the user can only reach whitelisted
addresses), and that claim is defeated by appending eleven characters to a
hostile URL. The `resource://rawfile/` test has the same shape and the same
weakness - `https://evil.com/#resource://rawfile/` passes it.

The fix is to **parse and compare the host**, and to compare it as a suffix
with an explicit dot (`host === 'huawei.com' || host.endsWith('.huawei.com')`)
so that `huawei.com.evil.com` fails and `consumer.huawei.com` passes. Match the
scheme prefix with `startsWith`, never with a search.

The normalisation step is not optional here: the seeded link resource is
`www.huawei.com` with no scheme, and `parseURL` throws on a scheme-less string.
A production version should reject anything that is not `https:` outright
rather than assuming it.

**Text into typed runs — `entry/src/main/ets/utils/UrlUtils.ets` and `components/CustomRichEditor.ets`** (as shipped)

```typescript
public static parseUrl(text: string): string[] {
  let urlPattern = /https?:\/\/(?:www\.)?\S+|(?<!\S)(?:\d{1,3}\.){3}\d{1,3}(?!\S)/g;
  let urls = text.match(urlPattern) || [];
  let parts = text.split(urlPattern);
  let result: string[] = [];
  for (let i = 0; i < parts.length; i++) {
    result.push(parts[i]);
    if (i < urls.length) {
      result.push(urls[i]);
    }
  }
  return result.filter(Boolean);
}

// CustomRichEditor.sendMessage()
private sendMessage() {
  let message: MsgTextImage[] = [];
  this.editorController?.getSpans().forEach(span => {
    let spanRes = span as RichEditorTextSpanResult;
    if (spanRes.textStyle !== undefined) {                 // a text span
      let spans = UrlUtils.parseUrl(spanRes.value);
      for (let i = 0; i < spans.length; i++) {
        message.push({
          type: UrlUtils.isUrl(spans[i]) ? MessageType.LINK : MessageType.TEXT,
          content: spans[i]
        });
      }
    } else {                                               // an image span
      message.push({
        type: MessageType.IMAGE,
        content: (span as RichEditorImageSpanResult).valueResourceStr
      });
    }
  });
  if (message.length > 0) {
    this.data.push({ isSelf: true, content: message });
  }
  this.editorController?.deleteSpans();
}
```

**Splitting by `match` plus `split` and interleaving is what preserves order.**
`String.split` with a regex returns the gaps, `match` returns the hits, and
zipping them back together reconstructs the sentence with each run tagged. The
`.filter(Boolean)` drops the empty strings `split` produces when a URL sits at
the start or the end.

Two traps in this code are worth naming. The `spanRes.textStyle !== undefined`
test is how you tell a text span from an image span in a `RichEditor` - there is
no discriminant field, so the presence of a style is the discriminator. And
`isUrl` re-tests each run with a **`/g`-flagged regex held in a fresh local**;
had that regex been hoisted to a module constant, `RegExp.lastIndex` would
persist between calls and alternate results on identical input. Keep `/g`
regexes local, or drop the flag on the test path.

The receiving side is one `Text` of `Span`s in `ChatPage`:

```typescript
Text() {
  ForEach(item.content, (content: MsgTextImage) => {
    if (content.type === MessageType.TEXT) {
      Span(content.content);
    } else if (content.type === MessageType.LINK) {
      Span(content.content)
        .fontColor('#1D63EF')
        .onClick(() => {
          this.pageInfos.pushPathByName('WebPage', content.content);
        });
    }
    // ... ImageSpan for MessageType.IMAGE
  }, (item: MsgTextImage, index: number) => `${JSON.stringify(item)}_${JSON.stringify(index)}`);
}
.constraintSize({ maxWidth: '75%' })
```

**The keyboard listener — `entry/src/main/ets/components/CustomRichEditor.ets`** (corrected, see `HW-14-0027`)

```typescript
import { KeyboardAvoidMode, window } from '@kit.ArkUI';
import { BusinessError } from '@kit.BasicServicesKit';

@State @Watch('onKeyboardChange') keyboardHeight: number = 0; // 软键盘高度
private currentWindow?: window.Window;                        // FIX: keep the window
private onKeyboardHeightChange: (data: number) => void =      // FIX: keep the callback
  (data: number) => {
    this.keyboardHeight = this.getUIContext().px2vp(data);
  };

aboutToAppear(): void {
  this.getUIContext().setKeyboardAvoidMode(KeyboardAvoidMode.RESIZE);
  window.getLastWindow(this.getUIContext().getHostContext()).then(currentWindow => {
    this.currentWindow = currentWindow;
    currentWindow.on('keyboardHeightChange', this.onKeyboardHeightChange);
  }).catch((err: BusinessError) => {                          // FIX: the sample drops the rejection
    LOGGER.error(`getLastWindow failed: ${err.code} ${err.message}`);
  });
}

aboutToDisappear(): void {                                    // FIX: absent in the sample
  this.currentWindow?.off('keyboardHeightChange', this.onKeyboardHeightChange);
}
```

**`on` without `off` is a leak with teeth.** The listener is registered per
component instance, and the window outlives every component in it, so each
mount adds a callback that keeps firing into a destroyed component for the rest
of the window's life. Here the chat page mounts once and the cost is bounded;
in `DropToSendImageAndText` the same code runs on every chat open, and in
`LatestMessage` it is the same again - three samples, one copy-pasted block,
which is why this is filed as a systematic (`HW-14-0027`).

Two mechanical requirements: `off` needs **the same function reference** that
`on` received, so the callback has to be stored rather than written inline; and
the `window` object arrives asynchronously, so it must be kept too or
`aboutToDisappear` has nothing to call `off` on.

The `@Watch('onKeyboardChange')` on `keyboardHeight` is the useful half of this
block: when the keyboard opens, the component calls the `keyboardHeightChange`
callback the parent passed in, and `ChatPage` uses it to scroll the list to the
end. The height itself only selects the bottom padding - 8vp with the keyboard
up, the navigation-indicator avoid area with it down.

## Permissions & config

```json5
"requestPermissions": [
  {
    "name": "ohos.permission.INTERNET",
    "reason": "$string:internet_reason",
    "usedScene": {
      "abilities": [
        "FormAbility"       // <- no such ability exists in this module
      ],
      "when": "inuse"
    }
  }
]
```

`ohos.permission.INTERNET` is `system_grant` and genuinely needed - the `Web`
component loads a remote page. But `usedScene.abilities` names `FormAbility`,
and this module declares exactly one ability, `EntryAbility`. It is a leftover
from the same copy-pasted template config that produced `HW-14-0003` (unused
`INTERNET`/`VIBRATE` declarations and dead location-permission constants in four
other social samples). The permission here is used; the scene that documents it
is fiction. Point `abilities` at `EntryAbility`.

`routerMap: "$profile:route_map"` maps the name `WebPage` to `webPageBuilder`,
so `ChatPage` pushes by name without importing the web page.
`resources/rawfile/blocked.html` is the interception page and must be shipped -
`loadUrl($rawfile('blocked.html'))` resolves against `rawfile`, and a missing
file leaves the user on a blank web view with no explanation.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- `onLoadIntercept` sees navigations the **web engine** initiates. It is not a
  network filter: a whitelisted page's own `fetch`/XHR to any host is not
  intercepted here, and neither are its subresources. If the requirement is
  "this view may only reach these hosts", combine it with
  `onInterceptRequest`.
- The whitelist is a single hardcoded token in `UrlUtils`. Anything
  configurable has to come from somewhere - and if it comes from the server, it
  must be fetched over a channel the whitelist itself does not gate.
- `javaScriptAccess(true)` is on. Do not relax the host check further while
  scripts are enabled.
- The seeded link string is `www.huawei.com` with no scheme, so the web engine
  picks the protocol. Store full `https://` URLs in real data.
- The message list is in-memory, seeded from the `MSG` constant; nothing is
  persisted and there is no network transport.
- `EntryAbility` registers `avoidAreaChange` and never releases it, and writes
  **raw px** into `AppStorage` (the pages convert with `px2vp`).

## Pitfalls

- **`HW-14-0026`** (B/medium, confirmed): `UrlUtils.canUrlAccess` accepts any
  URL containing the substring `www.huawei` or `resource://rawfile/` anywhere,
  so `https://evil.com/www.huawei` walks straight through the interception the
  sample exists to demonstrate. Fix: parse the URL and match the host as a
  suffix; test the scheme with `startsWith`.
- **`HW-14-0027`** (B/low, confirmed; systematic across three samples): the
  `keyboardHeightChange` listener is registered per component in
  `aboutToAppear` and never removed, and the `getLastWindow` promise has no
  `catch`. Listeners accumulate per open/rebuild and keep firing into destroyed
  components for the window's lifetime. Fix: store the callback and the window,
  and call `off` in `aboutToDisappear`. Same defect in
  `DropToSendImageAndText`'s `ChatDetails.ets` and `LatestMessage`'s
  `ChatPage.ets`.
- **`HW-14-0003`** (D/low, confirmed; systematic): copy-pasted permission
  configuration across the social samples. This sample is not in that finding's
  evidence list - its `INTERNET` declaration is genuinely used - but it carries
  the same template scar: `usedScene.abilities` names a `FormAbility` that does
  not exist in the module. Fix: name `EntryAbility`.
- **`HW-14-0001`** (E/low, confirmed; systematic): four social project trees
  list files their zips do not contain. This one's tree is accurate.
- Unfiled, worth knowing: `parseUrl`'s regex accepts a bare dotted quad as a
  URL, so `192.168.1.1` in a sentence becomes a tappable link; and the
  `ForEach` key is the stringified run plus its index, so it changes whenever
  the content does.

## References

- `huawei_industry_tree/14_social_communication/docs/12_h5_interception.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/h5_interception-0000002282125390
- `documentation/harmonyos-references/02_application-framework/arkts-basic-components-web.md` - the `Web` component, `javaScriptAccess`, `zoomAccess`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-basic-components-web
- `documentation/harmonyos-references/02_application-framework/arkts-basic-components-web-i.md` - `onLoadIntercept`, `OnLoadInterceptEvent`, `WebResourceRequest.getRequestUrl`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-basic-components-web-i
- `documentation/harmonyos-references/02_application-framework/arkts-apis-webview-backforwardlist.md` - `accessBackward` / `backward` and the history the back gesture walks
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-webview-backforwardlist
- `documentation/harmonyos-guides/04_system/permissions-for-all.md` - `ohos.permission.INTERNET` and `usedScene`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all
- `documentation/harmonyos-references/02_application-framework/js-apis-window.md` - `getLastWindow`, `on`/`off('keyboardHeightChange')`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-window
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-richeditor.md` - `getSpans`, text vs image span results
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-richeditor
- `documentation/harmonyos-guides/03_application-framework/arkts-navigation-jump.md` - `pushPathByName` and reading `pathInfo.param`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-navigation-jump
- `SOCIAL-11` - the same `RichEditor` input row, used for encryption instead
