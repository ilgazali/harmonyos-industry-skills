---
id: LIFE-28
title: Let an embedded H5 page pick a contact - javaScriptProxy bridging a local web page to the system contact picker
industry: 02_convenient_life
doc: huawei_industry_tree/02_convenient_life/docs/28_h5rechargeplatform.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/h5rechargeplatform-0000002387265697
sample: huawei_industry_tree/02_convenient_life/downloads/H5RechargePlateform.zip
kits: ["@kit.ArkWeb", "@kit.ContactsKit", "@kit.ArkUI", "@kit.AbilityKit", "@kit.PerformanceAnalysisKit"]
apis: [Web, "webview.WebviewController", javaScriptProxy, "webview.WebviewController.deleteJavaScriptRegister", domStorageAccess, fileAccess, geolocationAccess, onAlert, "$rawfile", "contact.selectContacts", "contact.ContactSelectionOptions", "contact.Contact", "contact.PhoneNumber", canIUse, StorageProp, "AppStorage.setOrCreate", "window.setWindowLayoutFullScreen", "window.getWindowAvoidArea", "window.on('avoidAreaChange')", "UIContext.px2vp", "UIContext.showAlertDialog"]
permissions: []
min_api: 24
modules: [entry]
findings: [HW-02-0235, HW-02-0236, HW-02-0237, HW-02-0238, HW-02-0239, HW-02-0240, HW-02-0241, HW-02-0242, HW-02-0243, HW-02-0244, HW-02-0245, HW-02-0246, HW-02-0269]
status: verified-with-fixes
---

## When to use

Load this card when **an H5 page has to reach a device capability**: a web form
that needs a contact, a photo, a scan result - anything the page cannot get on
its own. The recharge form here is the excuse; the bridge is the content.

The whole mechanism is one attribute and one object:

```
ArkTS side                                     H5 side (rawfile/index.html)
  class ContactsManagement {
    async selectContacts(): Promise<string>
  }
  Web({ src: $rawfile('index.html'), controller })
    .javaScriptProxy({
      object: this.contactsManagement,   ---->  window.contacts
      name: 'contacts',
      methodList: ['selectContacts'],    ---->  await contacts.selectContacts()
      controller: this.webViewController
    })
```

Two properties of this shape are worth internalising:

- **The proxy is registered with every frame of the page, iframes included.**
  The reference says so directly. Keep `src` local, keep `methodList` minimal,
  and never expose a method that does more than the page needs.
- **Registration has a matching teardown.** The reference states that
  `javaScriptProxy` "must be used together with the `deleteJavaScriptRegister`
  API to prevent memory leaks". The sample omits it (HW-02-0235).

**No permission is needed.** `contact.selectContacts` has no *Required
permissions* line in the reference - unlike `addContact` and its neighbours,
which all require `ohos.permission.WRITE_CONTACTS`. It is a system picker, so
the user chooses and only the chosen record crosses the boundary. That is why
this project's `module.json5` has no `requestPermissions` block.

## Feature checklist

- [ ] `deleteJavaScriptRegister` paired with the `javaScriptProxy`
      registration (HW-02-0235).
- [ ] `methodList` limited to what the page actually calls.
- [ ] `fileAccess` and `geolocationAccess` off unless needed.
- [ ] The contact result checked for an empty array **and** an empty
      `phoneNumbers` before either is indexed (HW-02-0236).
- [ ] The bridge method wrapped in try/catch so it cannot reject into the page
      (HW-02-0236), and the page's `await` wrapped too (HW-02-0241).
- [ ] The `canIUse` negative branch distinguishable from success
      (HW-02-0238).
- [ ] The phone number kept as a string end to end (HW-02-0240).
- [ ] `off('avoidAreaChange')` on teardown (HW-02-0237) and a `.catch` on
      `setWindowLayoutFullScreen` (HW-02-0239).

## Architecture

Three files, and the third is a web page.

| File | Role |
| --- | --- |
| `pages/Recharge.ets` | `@Entry`. The `Web` component, the proxy registration, and the `ContactsManagement` class exposed through it. |
| `resources/rawfile/index.html` | The recharge form: amount buttons, the phone field, the contacts icon, and the confirm and success dialogs. |
| `entryability/EntryAbility.ets` | Immersive layout and the two avoid-area heights. |

The bridged object is an ordinary class in the same file as the page:

```ts
class ContactsManagement {
  phoneNumber = '';

  async selectContacts(): Promise<string> {
    let contactSelectionOptions: contact.ContactSelectionOptions = { isMultiSelect: false };
    if (canIUse('SystemCapability.Applications.Contacts')) {
      // read the picked number
    }
    return this.phoneNumber;
  }
}
```

It is not `@Observed` and does not need to be - nothing in the ArkTS UI renders
from it. It exists purely as the far end of the bridge, and its return value is
what the web page awaits.

The `Web` component is locked down before the proxy is attached, which is the
right order to read it in:

```ts
.domStorageAccess(true)
.fileAccess(false)         // no local file access from the page
.geolocationAccess(false)  // no positioning from the page
.javaScriptProxy({ ... })  // one method, one name
```

`onAlert` is forwarded to a native dialog so the page's `alert()` calls surface
as system UI:

```ts
.onAlert((event) => {
  this.uiContext.showAlertDialog({ message: event?.message });
  return false;
})
```

Returning `false` lets the default handling continue; return `true` to suppress
it entirely.

## Implementation steps

Where the shipped code is wrong, the step gives the corrected version and names
the finding.

1. **Put the page in `rawfile` and load it with `$rawfile`.**

   ```ts
   Web({ src: $rawfile('index.html'), controller: this.webViewController })
   ```

   Local content is what makes the proxy safe to register - the method list is
   reachable only by the page you shipped.

2. **Close what the page does not need.**

   ```ts
   .domStorageAccess(true)
   .fileAccess(false)
   .geolocationAccess(false)
   ```

3. **Register the bridge.**

   ```ts
   .javaScriptProxy({
     object: this.contactsManagement,
     name: 'contacts',
     methodList: ['selectContacts'],
     controller: this.webViewController
   })
   ```

   Only one object can be registered this way; use
   `registerJavaScriptProxy` on the controller for more. The parameters cannot
   be updated after registration.

4. **Unregister it** (HW-02-0235). The sample has no teardown at all:

   ```ts
   aboutToDisappear(): void {
     this.webViewController.deleteJavaScriptRegister('contacts');
   }
   ```

   The name passed here is the `name` from the registration.

5. **Open the picker, and handle every ordinary outcome** (HW-02-0236,
   HW-02-0238):

   ```ts
   async selectContacts(): Promise<string> {
     if (!canIUse('SystemCapability.Applications.Contacts')) {
       throw new Error('contacts unavailable');       // shipped: returns '' silently
     }
     try {
       let options: contact.ContactSelectionOptions = { isMultiSelect: false };
       let contacts = await contact.selectContacts(options);
       let number = contacts?.[0]?.phoneNumbers?.[0]?.phoneNumber;
       this.phoneNumber = number ?? '';
     } catch (error) {
       let err = error as BusinessError;
       hilog.error(0x0000, 'Recharge', `selectContacts failed. Code: ${err.code}`);
       this.phoneNumber = '';
     }
     return this.phoneNumber;
   }
   ```

   Two things need guarding and the shipped one-liner guards neither: the
   returned array is empty when the user cancels, and `phoneNumbers` is an
   optional field that can also be an empty array on a contact with no stored
   number.

6. **Call it from the page, defensively** (HW-02-0241, HW-02-0240):

   ```javascript
   phoneBtn.addEventListener('click', async () => {
     try {
       const response = await contacts.selectContacts();
       if (!response) { return; }
       document.getElementById('phone').value = response.replace(/\D/g, '');
     } catch (e) {
       alert('无法获取联系人');
     }
   });
   ```

   Assign the cleaned **string**. The shipped page wraps it in `Number()`, which
   drops leading zeros.

7. **Set up the immersive layout - in the promise, with a catch**
   (HW-02-0239, HW-02-0237). The ordering in this sample is already right; only
   the rejection handler and the unsubscribe are missing:

   ```ts
   this.windowClass = windowStage.getMainWindowSync();
   this.windowClass.setWindowLayoutFullScreen(true).then(() => {
     let navArea = this.windowClass.getWindowAvoidArea(window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR);
     AppStorage.setOrCreate('bottomRectHeight', navArea.bottomRect.height);
     let sysArea = this.windowClass.getWindowAvoidArea(window.AvoidAreaType.TYPE_SYSTEM);
     AppStorage.setOrCreate('topRectHeight', sysArea.topRect.height);
     this.windowClass.on('avoidAreaChange', (data) => { /* later changes */ });
   }).catch((err: BusinessError) => {                       // shipped: no catch
     hilog.error(DOMAIN, 'testTag', 'setWindowLayoutFullScreen failed: %{public}s', JSON.stringify(err));
   });
   ```

   ```ts
   onWindowStageDestroy(): void {
     this.windowClass?.off('avoidAreaChange');
   }
   ```

8. **Pad the Web component with the stored heights.**

   ```ts
   .padding({
     top: this.uiContext.px2vp(this.topRectHeight),
     bottom: this.uiContext.px2vp(this.bottomRectHeight)
   })
   ```

   Here the two `@StorageProp` fields default to `0`, not `undefined`, which is
   the shape the sibling samples in this industry get wrong.

## Verified snippets

All snippets below are copied from the ZIP, not from the document.

`H5RechargePlateform.zip#entry/src/main/ets/pages/Recharge.ets:32-57` - the Web
component, its lockdown, the proxy and the alert forwarding:

```ts
  build() {
    Column() {
      Web({
        src: $rawfile('index.html'),
        controller: this.webViewController,
      })
        .domStorageAccess(true)
        .fileAccess(false)
        .geolocationAccess(false)
        .javaScriptProxy({
          // 配置ArkTS与H5的通信能力
          object: this.contactsManagement, // 注入ArkTS侧的对象实例
          name: 'contacts', // H5侧调用时使用的对象名称
          methodList: ['selectContacts'], // 暴露给H5调用的方法列表
          controller: this.webViewController  // 绑定当前Web组件的控制器
        })
        .onAlert((event) => {
          this.uiContext.showAlertDialog({ message: event?.message });
          return false;
        })
        .width('100%')
        .height('100%')
        .padding({
          top: this.uiContext.px2vp(this.topRectHeight),
          bottom: this.uiContext.px2vp(this.bottomRectHeight)
        });
```

`H5RechargePlateform.zip#entry/src/main/ets/pages/Recharge.ets:64-77` - the
bridged class in full:

```ts
class ContactsManagement {
  phoneNumber = '';

  // 选择联系人
  async selectContacts(): Promise<string> {
    // 选择联系人时的筛选条件 (是否多选)
    let contactSelectionOptions: contact.ContactSelectionOptions = { isMultiSelect: false };
    // 调用通讯录选择组件，让用户选择需要传入APP的通讯录联系人
    if (canIUse('SystemCapability.Applications.Contacts')) {
      this.phoneNumber = (await contact.selectContacts(contactSelectionOptions))[0].phoneNumbers?.[0].phoneNumber!;
    }
    return this.phoneNumber;
  }
}
```

The single expression on the tenth line carries HW-02-0236, and the missing
`else` carries HW-02-0238.

`H5RechargePlateform.zip#entry/src/main/resources/rawfile/index.html:219-225` -
the web side of the bridge:

```javascript
    phoneBtn.addEventListener('click', async () => {
        let response = await contacts.selectContacts()
        const cleanPhone = response.replace(/\D/g, '');
        document.getElementById('phone').value = Number(cleanPhone)

    });
```

Three lines, two findings: no try/catch around the bridge call
(HW-02-0241) and the `Number()` conversion of a phone number
(HW-02-0240).

`H5RechargePlateform.zip#entry/src/main/resources/rawfile/index.html:136` - the
field the number lands in, which is already the right type for a string:

```html
            <input type="tel" id="phone" placeholder="请输入手机号" maxlength="11"
```

`H5RechargePlateform.zip#entry/src/main/ets/entryability/EntryAbility.ets:41-54` -
the avoid-area reads inside the promise, which is the correct ordering:

```ts
    let windowClass: window.Window = windowStage.getMainWindowSync();
    // 1. 设置窗口全屏
    windowClass.setWindowLayoutFullScreen(true).then(() => {
      // 2. 获取布局避让遮挡的区域
      let type = window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR;
      let avoidArea = windowClass.getWindowAvoidArea(type);
      let bottomRectHeight = avoidArea.bottomRect.height;
      AppStorage.setOrCreate('bottomRectHeight', bottomRectHeight);

      type = window.AvoidAreaType.TYPE_SYSTEM;
      avoidArea = windowClass.getWindowAvoidArea(type);
      let topRectHeight = avoidArea.topRect.height;
      AppStorage.setOrCreate('topRectHeight', topRectHeight);
    });
```

This is the pattern `LIFE-24`, `LIFE-25` and `LIFE-27` all get wrong - the reads
belong inside the `.then()`. What is missing here is only the `.catch`
(HW-02-0239).

## Permissions & config

**No permissions.** `H5RechargePlateform.zip#entry/src/main/module.json5` has no
`requestPermissions` block. In the contact reference, `selectContacts` carries
no *Required permissions* line, while its neighbours `addContact`,
`deleteContact` and `updateContact` all list
`ohos.permission.WRITE_CONTACTS` - the picker is a system component and only the
record the user chooses crosses into the application.

Module configuration:

```json5
"deviceTypes": ["phone", "tablet", "2in1"],
"pages": "$profile:main_pages",
```

No `routerMap` - the sample is a single page.

Build target - **the highest in this industry**:

```json5
"compatibleSdkVersion": "6.1.1(24)",
"targetSdkVersion": "6.1.1(24)"
```

The H5 page and its three PNGs live in
`entry/src/main/resources/rawfile/`, which is what `$rawfile('index.html')`
resolves against.

## Constraints

- **API Version 24 and later**, DevEco Studio 6.1.1 Release and later. This is
  the only sample in the industry above API 20.
- **The proxy reaches every frame.** From the reference: "The object is
  registered with all frameworks of the web page, including all iframes, using
  the name specified in JavaScriptProxy."
- **`javaScriptProxy` must be paired with `deleteJavaScriptRegister`** to
  prevent memory leaks.
- **Only one object per `javaScriptProxy`.** Use
  `registerJavaScriptProxy` on the controller to register more.
- **The proxy parameters cannot be updated** after registration, and at least
  one of the synchronous and asynchronous method lists must be given.
- **`selectContacts` needs no permission**, and returns an array - empty when
  the user cancels.
- **`Contact.phoneNumbers` is optional** and may also be an empty array on a
  contact with no stored number.
- **`isMultiSelect: false`** still returns an array; the single selection is at
  index 0.
- **`getWindowAvoidArea` returns px**; this sample stores px and converts with
  `px2vp` at the point of use.

## Pitfalls

1. **HW-02-0235 - the proxy is registered and never unregistered.**
   `Recharge.ets:41-47` registers it; `deleteJavaScriptRegister` appears nowhere
   in the ZIP and the page has no `aboutToDisappear`. The Web attribute
   reference states the requirement in its note on this exact API: "The
   javaScriptProxy API must be used together with the deleteJavaScriptRegister9+
   API to prevent memory leaks." This is the one file a reader copies for the
   pattern, and it shows half of it.

2. **HW-02-0236 - the contact result is indexed twice with no checks.**
   `Recharge.ets:73` is
   `this.phoneNumber = (await contact.selectContacts(contactSelectionOptions))[0].phoneNumbers?.[0].phoneNumber!;`.
   Cancelling the picker gives an empty array, so `[0]` is undefined and reading
   `.phoneNumbers` off it throws; a contact with no stored number gives an empty
   `phoneNumbers`, so `[0].phoneNumber` throws. The optional chain guards the
   array being undefined but not either element, and the trailing `!` stops the
   compiler from raising any of it. Because the method is bridged, the throw
   surfaces inside the page's `await` with nothing logged on the ArkTS side.

3. **HW-02-0245 - the document reproduces that expression unchanged.** Step 2
   of `28_h5rechargeplatform.md` (`:42`) prints the same one-liner, with no try
   and no length check, while the contact reference's own example for the same
   API opens with `if (err) { console.error(...); return; }`. It is one of only
   two snippets in the document.

4. **HW-02-0238 - the unavailable-capability branch is indistinguishable from
   success.** `Recharge.ets:72` guards with
   `canIUse('SystemCapability.Applications.Contacts')` and has no `else`, so the
   method returns the initial empty string. The page then runs
   `Number('')`, which is `0`, and writes a zero into the phone field with no
   explanation.

5. **HW-02-0240 - the phone number is converted to a `Number`.**
   `index.html:222` is
   `document.getElementById('phone').value = Number(cleanPhone)`, into an
   `<input type="tel" maxlength="11">`. The regular expression on the previous
   line has already stripped everything non-numeric, so the conversion adds
   nothing and removes leading zeros - a stored landline like `01012345678`
   arrives as `1012345678`.

6. **HW-02-0241 - neither end of the bridge handles failure.**
   `index.html:220` awaits `contacts.selectContacts()` with no `try`, and
   `Recharge.ets:68-76` has none either. Combined with pitfall 2 that makes
   cancelling the picker a silent dead end: the field stays empty, no message
   appears, and nothing is logged.

7. **HW-02-0237 - `on('avoidAreaChange')` is never unsubscribed.**
   `EntryAbility.ets:57` subscribes on a local `windowClass` (`:41`);
   `onWindowStageDestroy` (`:76-79`) only logs.

8. **HW-02-0239 - `setWindowLayoutFullScreen` has a `then` and no `catch`.**
   `EntryAbility.ets:43-54`. Because both `AppStorage` writes live inside that
   `then`, a rejection also silently skips them, and the page renders with zero
   padding under the system bars. The reference notes the API "neither takes
   effect nor returns an error" in the freeform window state, and this module
   declares `2in1`.

9. **HW-02-0242 - two dialog buttons are reached through implicit element-id
   globals.** `index.html:201-208` declares eight elements with
   `getElementById`; `:244` and `:248` then use `cancelBtn` and `confirmBtn`
   with no declaration anywhere, working only because the ids are exposed as
   globals. Add the two missing lookups.

10. **HW-02-0243 - the success dialog never shows the number.**
    `index.html:207` looks up `successPhone` and nothing ever writes to it,
    while its sibling `successAmount` is set at `:233` in the same handler.

11. **HW-02-0244 - two `hilog` calls declare one identifier and pass two
    arguments.** `EntryAbility.ets:24-25` pass `'%{public}s'` with both a
    message and a serialised payload, so the payload - the launch parameters and
    the want - never reaches the log. The reference requires that "the number
    and type of parameters must map to the identifier in the format string".

12. **HW-02-0246 - the document swaps the industry's constraints section for an
    environment section and drops the SDK line.** `28_h5rechargeplatform.md:48-51`
    is headed 环境准备 with two bullets, while every sibling uses 约束与限制
    with three, the middle one naming the HarmonyOS SDK. This is the one sample
    in the industry that needs a newer API level - `6.1.1(24)` against
    `6.0.0(20)` everywhere else - and the one whose document does not say which
    SDK provides it.

## References

- Document:
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/h5rechargeplatform-0000002387265697
- Web attributes (`javaScriptProxy`, its iframe scope and the
  `deleteJavaScriptRegister` requirement):
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-basic-components-web-attributes
- WebviewController (`deleteJavaScriptRegister`, `registerJavaScriptProxy`):
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-webview-webviewcontroller
- @ohos.contact (`selectContacts`, `Contact`, `PhoneNumber`, and which APIs need
  `WRITE_CONTACTS`):
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/js-apis-contact
- Window (`setWindowLayoutFullScreen`, `getWindowAvoidArea`,
  `on`/`off('avoidAreaChange')`):
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-window
- hilog (format-argument mapping):
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-hilog
