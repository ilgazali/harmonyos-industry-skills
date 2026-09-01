---
id: SOCIAL-41
title: Receive a shared image - register a sendData skill, hand the Want to the page through LocalStorage
industry: 14_social_communication
doc: huawei_industry_tree/14_social_communication/docs/41_receive_image_share.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/receive_image_share-0000002405380589
sample: huawei_industry_tree/14_social_communication/downloads/ReceiveImageShare.zip
kits: ["@kit.AbilityKit", "@kit.ArkTS", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit", "@kit.ShareKit"]
apis: [Want, hilog, systemShare, window, "systemShare.getSharedData", getRecords, LocalStorage, "UIContext.getSharedLocalStorage", NavPathStack, pushPath, onPop, bindSheet]
permissions: []
min_api: 20
modules: [entry (entry)]
findings: [HW-14-0081, HW-14-0087]
status: verified-with-fixes
---

## When to use

**Load this card when your app has to appear in the system share sheet** and
turn what another app hands it into a message. The pattern has three moving
parts: a `skills` entry that makes the ability a share target for a declared set
of UTDs, a `Want` captured in the ability and republished to the page through
`LocalStorage`, and a `systemShare.getSharedData` parse that produces app-level
message objects.

The image case in this sample is the narrowest one. Nothing in the mechanism is
image-specific: swap `general.image` for `general.video`, `general.plain-text`
or `general.file` in the `uris` array and the same `getSharedData` /
`getRecords` loop reads them. That is why the document says other content types
can follow the same idea - the only per-type work is the extension whitelist and
the message renderer.

The part worth internalising is **where the Want lives**. A share launch can
either create the ability (`onCreate`) or re-target a running one (`onNewWant`),
and the page that acts on it is neither of those. This sample routes both
through one `LocalStorage` key and deletes it after use, which is the piece most
hand-rolled implementations get wrong - they read the Want once in `onCreate`
and a second share into a warm app does nothing.

## Feature checklist

- The app appears in the system share sheet when an image is shared from
  Gallery or any other app.
- Up to nine images can be shared in one action (`maxFileSupported: 9`).
- Launching by share opens the chat page and immediately pushes a
  "choose a chat" (选择聊天) page listing contacts.
- Picking a contact opens a bottom sheet previewing the shared image with
  cancel / send (发送) buttons.
- Send returns to the chat page and appends the image to that conversation as
  an outgoing (right-aligned) message.
- Sharing a non-image file shows a "该类型文件暂不支持" (this file type is not
  supported yet) toast and no navigation happens.
- A second share into the already-running app is handled, not ignored.
- Selecting any contact other than the first one toasts 仅支持转发给晶晶 - the
  demo forwards to one conversation only.

## Architecture

One `entry` module. A page pair joined by `Navigation` + `routerMap`, with the
share parsing pulled out into a module-level singleton.

```
entry/src/main/ets
├── component/MessageComp.ets       ListData: one bubble, text or image, mirrored by isSelf
├── entryability/EntryAbility.ets   captures the Want, publishes avoid areas + UIContext
├── model/ChatModel.ets             ChatContact (@ObservedV2/@Trace) + 7 seeded conversations
├── model/MessageModel.ets          MessageType, MsgTextImage, MsgContent
├── pages/ChatPage.ets              @Entry: the conversation, and the onPageShow share hook
├── pages/SelectChatPage.ets        NavDestination: contact list + the send sheet
└── utils/
    ├── Logger.ets                  hilog wrapper
    └── ShareReceiveUtil.ets        SHARE_RECEIVE_UTIL singleton: parse, hold, hand over
```

The documented tree matches the zip.

**The design decision worth copying** is that the share payload never travels as
a navigation parameter. `SHARE_RECEIVE_UTIL` is a module-level singleton that
holds the parsed URIs and a `hasShareReceive` flag; `ChatPage` only asks
`checkReceiveStatus()` whether to navigate, `SelectChatPage` asks
`getMsgContent()` for the message objects in `aboutToAppear`, and clears them in
`aboutToDisappear`. The `NavPathStack` carries only the contact list. That
split keeps a variable-length, possibly nine-item payload out of the route
params and gives the parse one owner with an explicit lifetime.

**The decision worth avoiding** is in the same file: that singleton also caches
`UIContext` in a field initializer (`HW-14-0081`). Module-level singletons are
constructed at import time, which is before the window stage has a UI context to
publish - so the field is permanently `undefined` and every toast it guards is
silently skipped. Hold data in a singleton; never hold UI handles.

## Implementation steps

1. **Declare the share skill** in `module.json5`: add
   `ohos.want.action.sendData` to the same `skills` entry as the home action,
   and enumerate every accepted type in `uris` with `scheme`, `utd` and
   `maxFileSupported`.
2. **Capture the Want in both entry points.** `onCreate` covers a cold share,
   `onNewWant` a share into a running app; both write the same
   `LocalStorage` key.
3. **Pass that `LocalStorage` into `loadContent`** so the page can read it -
   this is what makes `getSharedLocalStorage()` return anything.
4. **Read the Want in `onPageShow`, not `aboutToAppear`,** because a warm
   share re-shows an existing page instead of building a new one.
5. **Delete the key immediately after reading it,** otherwise every later
   foregrounding replays the same share.
6. **Parse with `getSharedData` + `getRecords`,** and validate the extension
   yourself - the UTD filter is the system's admission check, not a guarantee.
7. **Read the `UIContext` at call time, never in a field initializer**
   (`HW-14-0081`), or the unsupported-type toast never appears.
8. **Return the chosen messages through `pop(result)` / `onPop`,** appending
   them to the `@Trace` message array so the list re-renders.

## Verified snippets

All snippets are from `ReceiveImageShare.zip`. Corrected forms are marked.

**The share registration — `entry/src/main/module.json5`** (as shipped)

```json5
"skills": [
  {
    "entities": [
      "entity.system.home"
    ],
    "actions": [
      "action.system.home",
      "ohos.want.action.sendData"
    ],
    "uris": [
      {
        "scheme": "file",
        "utd": "general.image",
        "maxFileSupported": 9
      }
    ]
  }
]
```

**One `skills` entry carries both roles.** The home entity/action keep the app
in the launcher; `ohos.want.action.sendData` adds it to the share sheet. The
`uris` array is the admission filter: `utd: "general.image"` is the *abstract*
uniform data type, so every concrete image type below it (png, jpeg, heic …)
matches without being listed. `maxFileSupported: 9` is what makes the app show
up when the user has selected nine photos rather than one - a share of more
files than the declared maximum will not offer this app at all. No permission
is required anywhere in this flow: the URIs arrive already authorised for this
app by the share service.

**Capturing the Want in the ability — `entry/src/main/ets/entryability/EntryAbility.ets`** (as shipped)

```typescript
export default class EntryAbility extends UIAbility {
  storage: LocalStorage = new LocalStorage();

  onCreate(want: Want, launchParam: AbilityConstant.LaunchParam): void {
    this.context.getApplicationContext().setColorMode(ConfigurationConstant.ColorMode.COLOR_MODE_LIGHT);
    this.storage.setOrCreate('Want', want)
  }

  onNewWant(want: Want, launchParam: AbilityConstant.LaunchParam): void {
    this.storage.setOrCreate('Want', want)   // a share into an already-running app
  }

  onWindowStageCreate(windowStage: window.WindowStage): void {
    windowStage.loadContent('pages/ChatPage', this.storage, (err) => {
      if (err.code) {
        LOGGER.error(JSON.stringify(err));
        return;
      }
      this.setFullScreen(windowStage);
      AppStorage.setOrCreate('currentUIContext', windowStage.getMainWindowSync().getUIContext());
      windowStage.getMainWindowSync().getUIContext().setKeyboardAvoidMode(KeyboardAvoidMode.RESIZE)
    });
  }
}
```

**`loadContent(page, storage, callback)` is the load-bearing overload.** The
two-argument form does not attach a `LocalStorage`, and the page's
`getSharedLocalStorage()` then returns nothing. Writing the same key from both
`onCreate` and `onNewWant` is what makes the second share work: `onCreate` does
not run again for a warm launch.

Note the ordering trap here for later: `currentUIContext` is only written inside
the `loadContent` callback, well after module evaluation.

**Acting on the share — `entry/src/main/ets/pages/ChatPage.ets`** (as shipped)

```typescript
private pathStack: NavPathStack = new NavPathStack();
private storageProcess = this.getUIContext().getSharedLocalStorage()

async onPageShow(): Promise<void> {
  let want = this.storageProcess?.get('Want') as Want
  if (want === undefined) {
    return
  }
  await SHARE_RECEIVE_UTIL.getSharedData(want)
  this.storageProcess?.delete('Want') // 使用后及时清理want信息，避免重复使用。
  if (SHARE_RECEIVE_UTIL.checkReceiveStatus()) {
    this.pathStack.pushPath({
      name: 'SelectChatPage', param: this.chatContacts, onPop: (popInfo) => {
        let sendMsgs: MsgContent[] = popInfo.result as MsgContent[]
        sendMsgs.forEach((sendMsg: MsgContent) => {
          this.currentChat?.contactMsg.push(sendMsg)
        })
      }
    })
  }
}
```

**Three details carry the flow.** `onPageShow` rather than `aboutToAppear`,
because a warm share brings an existing page forward and `aboutToAppear` will
not fire again. The `delete('Want')` (the comment reads "clean up the want
promptly after use to avoid reusing it") is what stops every subsequent return
to the foreground from re-opening the picker with the previous share.
And the push is gated on `checkReceiveStatus()`, so an unparsable share leaves
the user on the chat page instead of on an empty picker.

The `onPop` callback pushes into `contactMsg`, which is a `@Trace` field of an
`@ObservedV2 ChatContact` - that is what makes the appended bubbles render
without any explicit state assignment on the page.

**The parse, and the toast that never fires — `entry/src/main/ets/utils/ShareReceiveUtil.ets`** (corrected, see `HW-14-0081`)

```typescript
const SUPPORT_FORMATS = ['png', 'jpg', 'jpeg', 'bmp', 'svg', 'webp', 'gif', 'heif', 'heic']

class ShareReceiveUtil {
  private hasShareReceive: boolean = false;
  private imageUri: string[] = []
  // FIX: shipped code is `uiContext: UIContext | undefined = AppStorage.get('currentUIContext');`
  //      evaluated at import time, before EntryAbility writes the key.

  private get uiContext(): UIContext | undefined {
    return AppStorage.get('currentUIContext');   // read lazily, at use time
  }

  async getSharedData(want: Want): Promise<void> {
    try {
      let shareData = await systemShare.getSharedData(want)
      let records = shareData.getRecords()
      if (records.length > 0) {
        records.forEach((record: systemShare.SharedRecord) => {
          this.parseRecord(record)
        });
      }
    } catch (error) {
      this.hasShareReceive = false;
      LOGGER.error(`Failed to getSharedData. Code: ${error.code}, message: ${error.message}`);
    }
  }

  parseRecord(record: systemShare.SharedRecord) {
    if (record.uri) {
      let ext = record.uri.split('.').reverse()[0].toLowerCase()
      for (let format of SUPPORT_FORMATS) {
        if (ext === format) {
          this.imageUri.push(record.uri)
          this.hasShareReceive = true;
          return;
        }
      }
      this.uiContext?.getPromptAction().showToast({ message: '该类型文件暂不支持' })
    } else {
      this.uiContext?.getPromptAction().showToast({ message: '该类型文件暂不支持' })
    }
  }
}

export const SHARE_RECEIVE_UTIL = new ShareReceiveUtil()
```

**The bug is a lifetime mismatch, and the optional chaining hides it.**
`export const SHARE_RECEIVE_UTIL = new ShareReceiveUtil()` runs the field
initializers the first time the module is imported - which is when `ChatPage` is
evaluated, before `loadContent`'s callback has published `currentUIContext`.
`AppStorage.get` therefore returns `undefined` for the lifetime of the process,
and `this.uiContext?.` quietly skips both toasts. Sharing a PDF does nothing
whatsoever: no message, no navigation, no log. Turning the field into a getter
(or simply calling `AppStorage.get` inside `parseRecord`) is the whole fix.

Note also that `getSharedData` swallows a failed parse into
`hasShareReceive = false`, so a thrown share is indistinguishable from no
share - acceptable here only because the page then stays put.

## Permissions & config

**None.** The sample declares no `requestPermissions`, and it does not need
any: the URIs delivered in the share `Want` come with a temporary grant for the
receiving app, so reading them requires neither `READ_IMAGEVIDEO` nor a picker.
This is the argument for receiving through the share sheet rather than opening a
gallery picker when the user already chose the files.

`routerMap` (`$profile:route_map`) registers `SelectChatPage` against
`selectChatPageBuilder`, which is why the push uses the bare name
`'SelectChatPage'` and no import of the destination component is needed in
`ChatPage`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- `deviceTypes` is `phone` only, and `supportWindowMode` is `fullscreen`.
- The demo forwards to one hardcoded conversation: `SelectChatPage` accepts
  only `index === 0` and toasts 仅支持转发给晶晶 for every other contact, and
  `ChatPage` always appends to `chatContacts[0]`.
- The send sheet previews `this.sendMsg[0]` only - a nine-image share is
  parsed and sent in full, but the sheet shows the first image.
- The chat input bar is inert: the `TextInput` is `focusable(false)` and the
  发送 button has no handler. Shared images are the only way to add a message.
- The extension whitelist is the real type filter. A file whose name lacks a
  dot takes `split('.').reverse()[0]` as the whole name and falls through to
  the unsupported branch, which is correct but only by accident.

## Pitfalls

- **`HW-14-0081`** (B/medium, confirmed): `ShareReceiveUtil` reads
  `AppStorage.get('currentUIContext')` in a field initializer that runs at
  module-evaluation time, while `EntryAbility` writes that key inside the
  `loadContent` callback; `uiContext` stays `undefined` and both
  该类型文件暂不支持 toasts are skipped by the optional chaining, so sharing an
  unsupported type produces no feedback at all. Fix: fetch the context from
  `AppStorage` inside `parseRecord` (or via a getter) instead of caching it.

## References

- `documentation/harmonyos-references/06_application-services/share-system-share.md` - `systemShare.getSharedData`, `SharedRecord`, `getRecords`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/share-system-share
- `documentation/harmonyos-guides/07_application-services/share-interface-description.md` - handling shared content inside an app
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/share-interface-description
- `documentation/harmonyos-guides/01_getting-started/module-configuration-file.md` - the `skills` tag, `uris`, `maxFileSupported`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/module-configuration-file
- `documentation/harmonyos-guides/03_application-framework/uniform-data-type-list.md` - the preset UTD list behind `general.image`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/uniform-data-type-list
- `documentation/harmonyos-references/02_application-framework/js-apis-app-ability-uiability.md` - `onCreate` vs `onNewWant`, `loadContent`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-app-ability-uiability
- `documentation/harmonyos-references/02_application-framework/ts-state-management.md` - `LocalStorage`, `AppStorage`, `@StorageProp`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-state-management
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-navigation.md` - `NavPathStack`, `pushPath`, `onPop`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-navigation
- `SOCIAL-39` - the sibling flow that inserts recent images into the same chat page
