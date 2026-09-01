---
id: COMMON-39
title: Privacy mode for an H5 login page - block screenshots from the web page through a JavaScript proxy
industry: 19_common_technical_solutions
doc: huawei_industry_tree/19_common_technical_solutions/docs/39_privacy_mode.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/privacy_mode-0000002376419961
sample: huawei_industry_tree/19_common_technical_solutions/downloads/H5SetPrivacy.zip
kits: ["@kit.ArkWeb", "@kit.ArkUI", "@kit.AbilityKit", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit"]
apis: ["Window.setWindowPrivacyMode", "window.getLastWindow", "Web.javaScriptProxy", "WebviewController.deleteJavaScriptRegister", "emitter.on", "emitter.off", "emitter.emit", "emitter.InnerEvent", "PromptAction.openCustomDialog", ComponentContent, "ComponentContent.update", SheetOptions, SheetSize, ScrollSizeMode, SheetDismiss, "SheetOptions.onHeightDidChange", "SheetOptions.shouldDismiss", "SheetOptions.onWillSpringBackWhenDismiss", "$rawfile", setTimeout, clearTimeout]
permissions: ["ohos.permission.PRIVACY_WINDOW"]
min_api: 20
modules: [entry]
findings: [HW-19-0117, HW-19-0118, HW-19-0119, HW-19-0182, HW-19-0183]
status: verified-with-fixes
---

## When to use

Load this card when a **sensitive screen rendered as an H5 page** - a login form,
a change-password form - must refuse screenshots and screen recording, and the
web page itself is what knows when the protection should be on.

Two mechanisms combine: `Window.setWindowPrivacyMode` provides the actual
anti-capture protection, and a **JavaScript proxy** lets the page turn it on and
off. A third piece - an `emitter`-based event bus - decouples the proxy object
from the page component that owns the window.

**Read HW-19-0117 before reusing.** Privacy mode is a *window* property, and the
sample never turns it back off.

## Feature checklist

The application must:

- Declare `ohos.permission.PRIVACY_WINDOW`.
- Register a proxy object into the H5 page exposing exactly the methods the page
  needs.
- Have the page request privacy mode as it loads.
- Apply the mode with `setWindowPrivacyMode` on the top window, handling both the
  callback error and the promise rejection (HW-19-0118).
- Suspend privacy mode while a legitimate full-screen overlay is showing (the
  agreement sheet), and restore it when that overlay shrinks or is dismissed.
- **Leave privacy mode when the protected page is dismissed** (HW-19-0117).
- Release the proxy, the event subscription and any pending timer on teardown.

## Architecture

Single-module project (`entry` HAP):

| File | Responsibility |
| --- | --- |
| `entryability/EntryAbility.ets` | window setup |
| `common/CommonConstants.ets` | `Constant.CALL_ARKTS_EVENT_ID`, `BIND_SHEET_READY_DELAY` and the layout numbers |
| `utils/WindowUtils.ets` | `setWindowPrivacyModeInPage(context, isFlag)` |
| `utils/CallArkTS.ets` | the object injected into the page - four methods, each of which only emits an event |
| `utils/PromptActionClass.ets` | opens, updates and closes the half-modal sheet |
| `utils/DisplayUtil.ets` | display metrics |
| `viewmodel/BindSheetData.ets` | `CallArkTSEventType`, `CallArkTSEventParams`, `BindSheetParams` |
| `component/BindSheetBuilder.ets` | the agreement / policy sheet content |
| `pages/SplashPage.ets`, `pages/LoginPage.ets`, `pages/NavigationPage.ets` | the three screens |
| `resources/rawfile/login.html`, `user_agreement.html`, `privacy_policy.html` | the pages |

**The bridge is deliberately thin.** `CallArkTS` does not touch the window at
all - every method builds a `CallArkTSEventParams` and emits it:

```ts
setPrivacy(isPrivacy: boolean) {
  let eventParams: CallArkTSEventParams = {
    eventType: CallArkTSEventType.SET_PRIVACY,
    params: { str: '', isPrivacy: isPrivacy }
  };
  this.sendCallArkTSEvent(eventParams);
}
```

`LoginPage` subscribes to that event id and dispatches on `eventType`. The
indirection is what lets the injected object be a plain class with no reference
to the page, the window or the UI context.

**`javaScriptProxy` rather than `registerJavaScriptProxy`.** The attribute form
binds the object when the component is created, and the page still releases it
explicitly in `aboutToDisappear` with
`deleteJavaScriptRegister('h5CallArkTSObjName')` - the pairing the reference
requires, and which COMMON-23 omits (HW-19-0073).

**Why the sheet has to suspend privacy mode.** A window in privacy mode blocks
capture of everything in it. When the user opens the user-agreement sheet to read
it, the sample lets it out of privacy mode - but only once the sheet has reached
full height, and after a short delay:

```ts
onHeightDidChange(height: number) {
  this.compareHeight(height);
  if (height === this.sheetHeight) {
    this.bindSheetTimeoutID = setTimeout(() => {
      this.setPrivacy(false);
    }, Constant.BIND_SHEET_READY_DELAY);
  } else if (height < this.sheetHeight) {
    this.setPrivacy(true);
  }
}

shouldDismiss(sheetDismiss: SheetDismiss) {
  this.setPrivacy(true);
  sheetDismiss.dismiss();
}
```

The delay exists so the login form underneath is fully covered before protection
drops; the `height < sheetHeight` branch re-engages it the moment the sheet is
dragged down.

## Implementation steps

1. **Declare `ohos.permission.PRIVACY_WINDOW`** in `module.json5`. It is
   system-grant, so no runtime request is needed.
2. **Write the window helper**: `window.getLastWindow(context)` then
   `setWindowPrivacyMode(flag, cb)`, handling the callback error **and** the
   promise rejection (HW-19-0118).
3. **Write the bridge object** as a plain class whose methods emit events - no
   UI or window references inside it.
4. **Attach it to the `Web`** with `javaScriptProxy({ object, name, methodList,
   controller })`, listing only the methods the page may call.
5. **Subscribe to the event id** in the page's `aboutToAppear` and dispatch on
   `eventType`.
6. **Have the page turn protection on as it loads**, from a `<script>` in its
   `<head>`.
7. **Suspend and restore around the sheet** using `onHeightDidChange` and
   `shouldDismiss`.
8. **Tear everything down** in `aboutToDisappear`:
   `deleteJavaScriptRegister`, `closeBindSheet`, `emitter.off`, `clearTimeout` -
   **and `setPrivacy(false)`** (HW-19-0117).

## Verified snippets

All snippets below come from the sample project, not from the document.

### The window helper (as shipped - see HW-19-0118)

`H5SetPrivacy.zip#H5SetPrivacy/entry/src/main/ets/utils/WindowUtils.ets`

```ts
export class WindowUtils {
  static setWindowPrivacyModeInPage(context: common.UIAbilityContext, isFlag: boolean) {
    window.getLastWindow(context).then((lastWindow) => {
      lastWindow.setWindowPrivacyMode(isFlag, (err: BusinessError) => {
        const errCode: number = err.code;
        if (errCode) {
          hilog.error(DOMAIN, TAG, 'Failed to set the window to privacy mode. 1Cause: %{public}s', JSON.stringify(err));
          return;
        }
        hilog.info(DOMAIN, TAG, 'Succeeded in setting the window to privacy mode.');
      });
    });
    // FIX (HW-19-0118): .catch((err: BusinessError) => hilog.error(...))
  }
}
```

### The bridge object

`H5SetPrivacy.zip#H5SetPrivacy/entry/src/main/ets/utils/CallArkTS.ets`

```ts
export class CallArkTS {
  setPrivacy(isPrivacy: boolean) {
    hilog.debug(DOMAIN, TAG, 'setPrivacy %{public}s', isPrivacy);
    let eventParams: CallArkTSEventParams = {
      eventType: CallArkTSEventType.SET_PRIVACY,
      params: { str: '', isPrivacy: isPrivacy }
    };
    this.sendCallArkTSEvent(eventParams);
  }

  userAgreementOnClick() { /* emits OPEN_USER_AGREEMENT */ }
  privacyPolicyOnClick() { /* emits OPEN_PRIVACY_POLICY */ }
  showToast(str: string)  { /* emits SHOW_TOAST */ }

  sendCallArkTSEvent(value: CallArkTSEventParams) {
    let event: emitter.InnerEvent = {
      eventId: Constant.CALL_ARKTS_EVENT_ID
    };
    let eventData: emitter.EventData = {
      data: { value: value }
    };
    // emitter.emit(event, eventData)
  }
}
```

### Injecting it into the page

`H5SetPrivacy.zip#H5SetPrivacy/entry/src/main/ets/pages/LoginPage.ets`

```ts
Web({ src: $rawfile('login.html'), controller: this.controller })
  .javaScriptProxy({
    object: this.h5CallArkTSObj,
    name: 'h5CallArkTSObjName',
    methodList: ['setPrivacy', 'userAgreementOnClick', 'privacyPolicyOnClick', 'showToast'],
    controller: this.controller,
  })
```

### The page requesting protection as it loads

`H5SetPrivacy.zip#H5SetPrivacy/entry/src/main/resources/rawfile/login.html`

```html
<script>
  h5CallArkTSObjName.setPrivacy(true);
</script>
```

Placed in `<head>`, so it runs before the form is painted.

### Dispatching the events

`H5SetPrivacy.zip#H5SetPrivacy/entry/src/main/ets/pages/LoginPage.ets`

```ts
let eventDialogCloseBtnOnclick: emitter.InnerEvent = {
  eventId: Constant.CALL_ARKTS_EVENT_ID
};
emitter.on(eventDialogCloseBtnOnclick, (eventData: emitter.EventData) => {
  if (eventData.data && eventData.data.value !== null && typeof eventData.data.value !== 'undefined' &&
    eventData.data.value !== undefined
  ) {
    let eventParams: CallArkTSEventParams = eventData.data.value;
    if (eventParams.eventType === CallArkTSEventType.SET_PRIVACY) {
      this.setPrivacy(eventParams.params.isPrivacy);
    } else if (eventParams.eventType === CallArkTSEventType.OPEN_USER_AGREEMENT ||
      eventParams.eventType === CallArkTSEventType.OPEN_PRIVACY_POLICY) {
      this.bindSheetOnClick(eventParams.eventType);
    } else if (eventParams.eventType === CallArkTSEventType.SHOW_TOAST) {
      this.showToast(eventParams.params.str);
    }
  }
});
```

### Suspending around the sheet

`H5SetPrivacy.zip#H5SetPrivacy/entry/src/main/ets/pages/LoginPage.ets`

```ts
this.options = {
  detents: [SheetSize.LARGE, SheetSize.MEDIUM],
  title: {
    title: value === CallArkTSEventType.OPEN_USER_AGREEMENT ? $r('app.string.title_user_agreement') :
      $r('app.string.title_privacy_policy')
  },
  scrollSizeMode: ScrollSizeMode.CONTINUOUS,
  showClose: true,
  onHeightDidChange: (height: number) => {
    this.onHeightDidChange(height);
  },
  shouldDismiss: (sheetDismiss: SheetDismiss) => {
    this.shouldDismiss(sheetDismiss);
  },
  onWillSpringBackWhenDismiss: () => {
  },
};
PromptActionClass.setOptions(this.options);
PromptActionClass.openBindSheet();
```

### Teardown (as shipped - see HW-19-0117)

`H5SetPrivacy.zip#H5SetPrivacy/entry/src/main/ets/pages/LoginPage.ets`

```ts
aboutToDisappear(): void {
  try {
    this.controller.deleteJavaScriptRegister('h5CallArkTSObjName');
  } catch (error) {
    let message = (error as BusinessError).message;
    let code = (error as BusinessError).code;
    hilog.error(DOMAIN, TAG,
      'deleteJavaScriptRegister error code is %{public}d, message is %{public}s',
      code, message);
  }

  PromptActionClass.closeBindSheet();
  emitter.off(Constant.CALL_ARKTS_EVENT_ID);
  clearTimeout(this.bindSheetTimeoutID);
  // FIX (HW-19-0117): this.setPrivacy(false);
}
```

Everything else this page allocated is released here - the proxy, the sheet, the
event subscription and the pending timer. Only the window mode is left on.

## Permissions & config

`ohos.permission.PRIVACY_WINDOW` is required and is the only permission the
document lists and the sample declares.

`H5SetPrivacy.zip#H5SetPrivacy/entry/src/main/module.json5`:

```json5
"requestPermissions": [
  {
    "name": "ohos.permission.PRIVACY_WINDOW"
  }
]
```

It is a system-grant permission - the declaration is enough, there is no runtime
dialog. Note what is **not** declared: no `ohos.permission.INTERNET`, correctly,
because all three HTML pages are packaged `$rawfile`s.

## Constraints

- **API level.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later. `setWindowPrivacyMode` is an
  API 9 interface.
- **Privacy mode is a window property, not a page property.** It stays in force
  until something clears it, across page navigations - which is the substance of
  HW-19-0117.
- **`javaScriptProxy` must be paired with `deleteJavaScriptRegister`**, which this
  sample does correctly.
- **The proxy is exposed to all page frames.** All three pages here are packaged
  rawfiles, so the exposure is contained; injecting the same object into a remote
  page would hand that page the ability to disable the screenshot protection.
- **The page decides when protection engages.** If `login.html` fails to load or
  its `<head>` script throws, `setPrivacy(true)` never runs and the form is
  capturable - so the protection is only as reliable as the page load.
- **`onHeightDidChange` fires repeatedly during the sheet drag.** The equality
  test `height === this.sheetHeight` is what distinguishes "fully open" from
  intermediate positions; the timer it starts must be cancelled on teardown,
  which the sample does.
- **Devices.** Per `module.json5`.

## Pitfalls

- **Privacy mode is never turned off, which is incorrect.** Only the
  fully-expanded agreement sheet clears it; leaving the login page does not, so
  the whole application - including every page after sign-in - keeps refusing
  screenshots for the rest of the session. Add `setPrivacy(false)` to
  `aboutToDisappear`. (HW-19-0117)
- **`getLastWindow` has no rejection handler, which is incorrect.** A rejection
  leaves the anti-capture protection off with nothing in the log, so a failed
  security control looks exactly like a working one. (HW-19-0118)
- **The documented project tree misspells two directories** - `components` for
  the shipped `component`, and `entrybackupablility` for `entrybackupability` -
  in a project that builds with `caseSensitiveCheck` enabled. (HW-19-0119)
- **Protection depends on page JavaScript.** Engaging privacy mode from
  `login.html` means a page that fails to load leaves the screen unprotected;
  engaging it from the page component's `aboutToAppear` and letting the page only
  *suspend* it would fail closed instead.
- **The `emitter` event id is global to the application.** Any other component
  emitting `Constant.CALL_ARKTS_EVENT_ID` reaches this handler, including one that
  could pass `isPrivacy: false`.
- **The log message contains a stray marker** - `'Failed to set the window to
  privacy mode. 1Cause: %{public}s'` - which will make the line hard to grep for.

## References

- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-window.md` -
  `setWindowPrivacyMode` and its callback/promise forms.
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/arkts-apis-window-window#setwindowprivacymode9
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-f.md` -
  `window.getLastWindow` and its `.catch` example.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-f
- `documentation/harmonyos-references/02_application-framework/arkts-basic-components-web-attributes.md` -
  the `javaScriptProxy` attribute.
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/arkts-basic-components-web-attributes#javascriptproxy
- `documentation/harmonyos-references/02_application-framework/arkts-apis-webview-webviewcontroller.md` -
  `deleteJavaScriptRegister` and the pairing requirement.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-webview-webviewcontroller
- `documentation/harmonyos-guides/03_application-framework/web-in-page-app-function-invoking.md` -
  the front-end-calls-application-function pattern the document links.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/web-in-page-app-function-invoking
- `documentation/harmonyos-references/03_system/js-apis-emitter.md` - `emitter.on`
  / `off` / `emit` and `InnerEvent`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-emitter
- https://developer.huawei.com/consumer/cn/doc/harmonyos-references/ts-universal-attributes-sheet-transition -
  `SheetOptions`, `detents`, `onHeightDidChange`, `shouldDismiss`.
- https://developer.huawei.com/consumer/cn/doc/harmonyos-faqs/faqs-arkui-3 - the
  anti-screenshot FAQ the document links.
- https://developer.huawei.com/consumer/cn/doc/architecture-guides/privacy_mode-0000002376419961
