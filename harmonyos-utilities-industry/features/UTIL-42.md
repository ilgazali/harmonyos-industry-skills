---
id: UTIL-42
title: HCE app link - emulate an NFC tag whose NDEF record is a URI, and read it back with openLink
industry: 15_utilities
doc: huawei_industry_tree/15_utilities/docs/42_hce_link_app.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/hce_link_app-0000002550216593
sample: huawei_industry_tree/15_utilities/downloads/HceLinkApp.zip
kits: ["@kit.AbilityKit", "@kit.ArkTS", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.ConnectivityKit", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: [Want, buffer, bundleManager, cardEmulation, common, hilog, nfcController, tag, window]
permissions: [ohos.permission.NFC_TAG, ohos.permission.NFC_CARD_EMULATION]
min_api: 20
modules: [entry (entry)]
findings: [HW-15-0008, HW-15-0085, HW-15-0086, HW-15-0087, HW-15-0101, HW-15-0102]
status: verified-with-fixes
---

## When to use

**Load this card when one phone has to behave like an NFC tag and another has
to read it** - the 碰一碰 (tap-to-pair) shape: tap the two devices together and
the reader opens a deep link into an app. The sample implements both halves in
one app, on two tabs: a 模拟卡片 (emulate card) tab that host-card-emulates a
tag carrying an NDEF URI record, and a 读卡 (read card) tab that runs
foreground reader mode, sends a SELECT AID and opens whatever URI comes back.

The transferable structure is the pairing of the two APIs.
`cardEmulation.HceService` is the *card* side: register an AID, subscribe to
`hceCmd`, answer each APDU with `transmit`. `tag.on('readerMode', ...)` is the
*reader* side: claim NFC in the foreground, get a `TagInfo` per discovered tag,
pick a technology, connect, exchange. Anything else built on NFC in an app -
an access badge, a loyalty card, a device-onboarding tap - is one of these two
with different bytes.

**Read `HW-15-0087` before adopting the HCE half.** The subscription lifecycle
in this sample is wrong in a way that multiplies with every tab switch, and
the APDU response is not a conformant ISO 7816 response.

## Feature checklist

- On `onCreate`, the app checks `canIUse('SystemCapability.Communication.NFC.Core')`
  and `cardEmulation.hasHceCapability()` before touching any NFC API.
- A two-tab home page (read card / emulate card) with no swipe (`scrollable(false)`).
- Emulate tab: an 应用跳转 (app link) row opens a sheet with a text field; the
  entered URL - or a default from resources when empty - is encoded as an NDEF
  URI record and kept as the bytes the card will answer with.
- Entering the emulate tab starts `HceService` with a registered AID and
  subscribes to `hceCmd`; leaving it stops the service.
- Read tab: entering it enables foreground reader mode for NFC-A / NDEF /
  MIFARE Ultralight; leaving it disables it.
- On a tag read, the page shows 状态, ID (colon-separated hex UID), 技术类型,
  SAK, ATQA, 是否可写 (writable) and 只为可读 (can set read-only), plus a list
  of decoded NDEF records.
- The reader sends a SELECT AID command over IsoDep, parses the response as an
  NDEF message and calls `openLink(url, { appLinkingOnly: true })`.

## Architecture

One `entry` module. A singleton util holds all NFC state; the pages are views
over it.

```
entry/src/main/ets
├── common/GlobalContext.ets         static UIAbilityContext holder (for openLink from the util)
├── constants
│   ├── CommonConstant.ets           CMD_DATA (the SELECT AID APDU) and aidStr
│   └── Constant.ets                 font sizes/weights for the title rows
├── entryability/EntryAbility.ets    canIUse + hasHceCapability, HceService creation, avoid areas
├── entrybackupability/
├── model
│   ├── NdefTagModel.ets             NDEF encode/decode: URI, text, external records
│   └── NfcATagModel.ets             SAK / ATQA hex formatting for an NfcATag
├── pages
│   ├── Index.ets                    @Entry: the two tabs and their on/off lifecycle
│   ├── HcePage.ets                  the emulate tab: sheet + TextInput -> recordBytes
│   └── ReadCardPage.ets             the read tab: the card-info list and the record list
└── utils
    ├── NfcUtil.ets                  singleton: reader mode, HCE service, readerModeCb
    └── UriPrefixUtil.ets            the NDEF URI prefix table (0x01 -> http://www. etc.)
```

**The documented tree does not match the zip**: the doc's 工程目录 lists
`utils/NfcUtils.ets`, the zip ships `utils/NfcUtil.ets` (singular), and every
import in the sample uses the singular form (`HW-15-0008`). The doc also spells
the backup directory `entrybackupablility`.

**The design decision worth copying** is `NfcUtil` as an `@Observed` singleton
that the pages hold with `@State nfcUtil: NfcUtil = NfcUtil.getInstance()`.
NFC callbacks do not arrive on a component - `tag.on('readerMode', ...)` fires
against an `ElementName`, and `HceService.on('hceCmd', ...)` against the
service - so there is no `this` to write into. Making the shared holder
observed means the callback writes plain fields (`uidHexString`, `nfcA`,
`ndef`) and the list rows re-render, without any event bus.

The trap in that decision is visible in `readerModeCb`: because it is passed
as a bare function reference (`NfcUtil.getInstance().readerModeCb`), `this` is
not bound inside it, and the body has to say `NfcUtil.getInstance().isIdentified = true`
everywhere. That works, and it is also why the method silently mixes the
instance and static access styles (`NfcUtil.instance` in `hceCommonTransmit`,
`NfcUtil.getInstance()` in `readerModeCb`).

## Implementation steps

1. **Gate on capability before anything else.** `canIUse` for the NFC syscap,
   `cardEmulation.hasHceCapability()` for HCE. Both return, not throw, on a
   device without the hardware.
2. **Build the two `ElementName`s from the launch `want`** in `onCreate` -
   `bundleName`, `abilityName`, `moduleName`. Both `tag.on` and
   `HceService.start` identify the foreground app by element, not by context.
3. **Declare the two skill actions** in `module.json5`:
   `ohos.nfc.tag.action.TAG_FOUND` and
   `ohos.nfc.cardemulation.action.HOST_APDU_SERVICE`. Without them the system
   will not route taps to this ability.
4. **Check `nfcController.isNfcOpen()` on every entry** to a tab; NFC can be
   switched off while the app is running.
5. **Pair every `on` with an `off`.** `nfcTagOn`/`nfcTagOff` are paired
   correctly; `hceServiceOn` has no counterpart, so the `hceCmd` subscription
   accumulates on each tab show (`HW-15-0087`).
6. **Encode the link as an NDEF URI record** with `tag.ndef.makeUriRecord`,
   wrap it with `createNdefMessage` and flatten with `messageToBytes` - never
   hand-roll the prefix byte.
7. **Answer APDUs per command, with a status word.** Branch on the incoming
   command (SELECT AID vs READ BINARY) and append `0x90 0x00`
   (`HW-15-0087`); a bare NDEF payload is not a valid response.
8. **Return from the error branch of `readerModeCb`** instead of falling
   through into the technology loop (`HW-15-0085`).
9. **Construct `NdefTagModel` when the tag exposes NDEF**, or the writable /
   read-only rows and the whole record list stay empty (`HW-15-0086`).

## Verified snippets

All snippets are from `HceLinkApp.zip`. Corrected forms are marked.

**The reader-mode subscription - `entry/src/main/ets/utils/NfcUtil.ets`** (as shipped)

```typescript
const TECH_LIST: number[] = [tag.NFC_A, tag.NDEF, tag.MIFARE_ULTRALIGHT]; // 仅演示NFC_A类型

public nfcTagOn(callback: AsyncCallback<tag.TagInfo, void>) {
  if (this.nfcTagElementName !== undefined) {
    // 调用tag模块中前台优先的接口，使能前台应用程序优先处理所发现的NFC标签功能
    try {
      tag.on('readerMode', this.nfcTagElementName, TECH_LIST, callback);
      this.foregroundRegister = true;
    } catch (error) {
      hilog.error(0x0000, 'testTag', 'on readerMode error = %{public}s', JSON.stringify(error));
    }
  }
}

public nfcTagOff() {
  // 退出应用程序NFC标签页面时，调用tag模块退出前台优先功能
  if (this.foregroundRegister) {
    this.foregroundRegister = false;
    this.isIdentified = false;
    try {
      tag.off('readerMode', this.nfcTagElementName);
    } catch (error) {
      hilog.error(0x0000, 'testTag', 'off readerMode error = %{public}s', JSON.stringify(error));
    }
  }
}
```

**This is the pair to copy.** `readerMode` is *foreground priority*: while it
is on, this app receives every discovered tag instead of the system dispatch,
so leaving it on after the page goes away hijacks NFC for the whole device.
The `foregroundRegister` flag makes `off` idempotent - `tag.off` on an
unregistered element throws, and the tab lifecycle can fire `onWillHide`
without a matching `onWillShow` (the first tab is shown by `aboutToAppear`,
not by `onWillShow`). `TECH_LIST` is the filter: only tags exposing NFC-A,
NDEF or MIFARE Ultralight raise the callback.

The HCE half has no such pair. `hceServiceOn` calls `start` and
`hceService.on('hceCmd', callback)`; nothing anywhere calls
`hceService.off('hceCmd', ...)`.

**The reader callback - same file** (corrected, see `HW-15-0085`, `HW-15-0086`)

```typescript
public async readerModeCb(error: BusinessError, tagInfo: tag.TagInfo) {
  if (error) {
    hilog.info(0x0000, 'testTag', 'readerModeCb error %{public}s', JSON.stringify(error));
    return;                                     // FIX: the sample only logs and falls through
  }
  if (tagInfo === null || tagInfo === undefined) {
    hilog.error(0x0000, 'testTag', 'readerModeCb tagInfo is invalid');
    return;
  }
  if (tagInfo.technology === null || tagInfo.technology === undefined || tagInfo.technology.length === 0) {
    hilog.error(0x0000, 'testTag', 'readerModeCb technology is invalid');
    return;
  }
  NfcUtil.getInstance().isIdentified = true;
  NfcUtil.getInstance().uidHexString = getTagInfoUid(tagInfo);
  NfcUtil.getInstance().techListString = getTagInfoTech(tagInfo);

  let nfcATag: tag.NfcATag | null = null;
  for (let i = 0; i < tagInfo.technology.length; i++) {
    if (tagInfo.technology[i] === tag.NFC_A) {
      nfcATag = tag.getNfcA(tagInfo);
    }
    if (tagInfo.technology[i] === tag.NDEF) {
      // FIX: absent in the sample - without this, ndef stays undefined forever
      NfcUtil.getInstance().ndef = new NdefTagModel(tag.getNdef(tagInfo));
    }
  }
  if (nfcATag !== null) {
    nfcATag.connect();
    if (nfcATag.isConnected()) {
      NfcUtil.getInstance().nfcA = new NfcATagModel(nfcATag);
    }
  }
  // ... IsoDep: connect, transmit CMD_DATA, decode the NDEF message, openLink
}
```

**Two defects live in the shipped shape of this function.** The success path is
one big `if (!error) { ... }` block containing every null guard, and the `else`
branch only logs - execution then continues past the block into
`for (let i = 0; i < tagInfo.technology.length; i++)` for the IsoDep scan.
On a callback error `tagInfo` is not guaranteed to be an object, so the error
path ends in a `TypeError` rather than a clean return (`HW-15-0085`). Inverting
the guard - `if (error) return;` - removes the whole class of problem and
un-nests the body.

The second: `NdefTagModel` is a complete, working class - it reads
`isNdefWritable()`, `canSetReadOnly()` and every record, and decodes URI, text
and external payloads - and it is **never constructed** in the reader path.
`NfcUtil.ndef` is declared and stays `undefined`, so `ReadCardPage` renders
是否可写 and 只为可读 as the literal string `"undefined"` (they are built with
a template literal, `${this.nfcUtil.ndef?.isNdefWritable}`, so the optional
chain does not save them) and `ForEach(this.nfcUtil.ndef?.recordContents, ...)`
iterates nothing (`HW-15-0086`). Half of the documented read page is dead.

**The card-side response - same file** (corrected, see `HW-15-0087`)

```typescript
public hceCommonTransmit(error: BusinessError, hceCommand: number[]): void {
  if (error) {
    hilog.error(0x0000, 'testTag', 'hceCommonTransmit error %{public}s', JSON.stringify(error));
    return;
  }
  if (hceCommand == null) {
    return;
  }
  const self = NfcUtil.getInstance();
  if (self.hceService === undefined) {
    return;
  }
  // FIX: the sample tests `recordBytes === undefined` on a field initialised to []
  if (self.recordBytes.length === 0) {
    hilog.info(0x0000, 'testTag', 'no record written yet');
    self.hceService.transmit([0x6A, 0x82]);     // FIX: file not found, instead of an empty payload
    return;
  }
  // FIX: the sample transmits the raw NDEF bytes for every command, with no status word
  const response: number[] = self.recordBytes.concat([0x90, 0x00]);  // SW1SW2 = success
  self.hceService.transmit(response)
    .catch((err: BusinessError) => {
      hilog.error(0x0000, 'testTag', 'hceService transmit error = %{public}s', JSON.stringify(err));
    });
}
```

**An APDU response is a payload followed by a two-byte status word.** The
shipped code answers every command - including the reader's SELECT AID, which
expects only `90 00` - with the full NDEF message and no SW1SW2. A permissive
reader (this sample's own read tab, which parses whatever comes back as an NDEF
message) works; a standards-conformant one rejects the response as malformed.
The real card applet branches on the command: `00 A4 04 00` (SELECT by name)
answers `90 00`, `00 B0` (READ BINARY) answers the data plus `90 00`.

The `recordBytes === undefined` guard is dead code: the field is declared
`public recordBytes: number[] = []`, so it is never undefined, and before the
user has written anything the card transmits an **empty array** - the reader
then tries `createNdefMessage([])` and fails opaquely.

**The subscription lifecycle - `entry/src/main/ets/pages/Index.ets`** (corrected, see `HW-15-0087`)

```typescript
TabContent() {
  HcePage();
}
.onWillShow(() => {
  if (!NfcUtil.getInstance().isNfcOpen()) {
    hilog.info(0x0000, 'testTag', 'nfc is not open');
    return;
  }
  NfcUtil.getInstance().hceServiceOn(NfcUtil.getInstance().hceCommonTransmit);
})
.onWillHide(() => {
  const util = NfcUtil.getInstance();
  if (util.hceElementName === undefined || util.hceService === undefined) {
    return;
  }
  try {
    const hceService = util.hceService as cardEmulation.HceService;
    hceService.off('hceCmd', util.hceCommonTransmit);   // FIX: absent in the sample
    hceService.stop(util.hceElementName);
  } catch (error) {
    hilog.error(0x0000, 'testTag', 'hceService.stop error = %{public}s', JSON.stringify(error));
  }
})
```

**`stop()` is not `off()`.** `stop` deregisters the AID so the app stops being
selected; the `hceCmd` subscription made in `hceServiceOn` survives it. Every
return to the emulate tab therefore adds another callback to the same service,
and on the next APDU **all** of them fire, each calling `transmit` - N
responses for one command. The reader sees the first and the rest error out.
The same page also subscribes from `aboutToAppear` when `currentIndex` is 1, so
the initial state can already be doubled.

Note `.scrollable(false)` on the `Tabs`: with an NFC session tied to tab
visibility, a half-finished swipe that fires `onWillShow` and then reverses is
a real hazard. Disabling the swipe and switching only from the bar makes the
show/hide pairs deterministic - worth copying whenever a tab owns hardware.

## Permissions & config

```json5
"requestPermissions": [
  { "name": "ohos.permission.NFC_TAG", "reason": "$string:NFC_reason" },
  { "name": "ohos.permission.NFC_CARD_EMULATION", "reason": "$string:NFC_reason" }
],
"abilities": [{
  "skills": [{
    "entities": ["entity.system.home"],
    "actions": [
      "action.system.home",
      "ohos.want.action.home",
      "ohos.nfc.tag.action.TAG_FOUND",
      "ohos.nfc.cardemulation.action.HOST_APDU_SERVICE"
    ]
  }]
}]
```

- Both permissions are `system_grant` at `normal` level, so there is no runtime
  request anywhere in the sample - declaring them is enough.
- `usedScene` is omitted, which is allowed for system_grant permissions;
  `reason` is present for both, pointing at the same string resource.
- The two NFC **actions** are the load-bearing part. `TAG_FOUND` lets the
  system deliver a discovered tag to this ability; `HOST_APDU_SERVICE` marks it
  as an HCE host. Adding them to the home skill (as here) means one ability
  handles launch, tap and card emulation.
- The AID is `A0000000031010` and the reader's SELECT command in
  `CommonConstant.CMD_DATA` carries the same seven bytes. Change one and you
  must change the other.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **Two devices with NFC hardware are needed**, and neither half runs on the
  emulator. `hasHceCapability()` is false on most non-Huawei hardware.
- The doc notes 模拟卡片中填写的跳转应用需预先安装 - the app the link points
  at must already be installed, because `openLink` is called with
  `appLinkingOnly: true`, which refuses to fall back to a browser.
- `GlobalContext` holds the `UIAbilityContext` in a static field so
  `readerModeCb` can call `openLink` from outside a component. It is never
  cleared on `onDestroy`.
- `EntryAbility` calls `setWindowLayoutFullScreen(true)` twice - once bare,
  once with a promise chain - and registers `avoidAreaChange` without ever
  releasing it.
- The emulate tab writes exactly one record type (URI). `NdefTagModel` decodes
  text and external-type records on read, but there is no UI to write them.

## Pitfalls

- **`HW-15-0008`** (E/low, confirmed): the doc's 工程目录 lists
  `utils/NfcUtils.ets`; the zip ships `utils/NfcUtil.ets`. Fix: correct the
  tree entry to the singular name.
- **`HW-15-0085`** (B/medium, confirmed): the `else` branch of
  `readerModeCb`'s `if (!error)` only logs, then execution falls through into
  `for (... tagInfo.technology.length ...)` - but the null guards live inside
  the success branch, so a callback error becomes a `TypeError` instead of a
  clean exit. Fix: `return` in the error branch (or invert the guard).
- **`HW-15-0086`** (B/medium, confirmed): `NfcUtil.ndef` is declared and never
  assigned. `readerModeCb` builds only `nfcA`, so 是否可写 and 只为可读 render
  as `"undefined"` and the NDEF record list is always empty, although
  `NdefTagModel` is fully implemented. Fix: construct `NdefTagModel` when the
  tag's technology list contains `tag.NDEF`.
- **`HW-15-0087`** (B/medium, confirmed): `hceCmd` is subscribed on every
  `onWillShow` of the emulate tab and never unsubscribed - `onWillHide` calls
  only `hceService.stop()` - so N tab visits mean N callbacks and N `transmit`
  calls per APDU. The response itself is the raw NDEF payload with no ISO 7816
  status word and no branching on the received command, and the
  `recordBytes === undefined` guard is dead against a field initialised to
  `[]`, so the first run transmits an empty array. Fix: `off` in `onWillHide`,
  append `0x90 0x00`, and branch on the command.
- **`this` is unbound in the callbacks** — both `readerModeCb` and
  `hceCommonTransmit` are passed as bare references and reach state through
  `NfcUtil.getInstance()` / `NfcUtil.instance`. Keep it that way or bind
  explicitly; a refactor to `this.` compiles and then reads `undefined`.

## References

- `documentation/harmonyos-references/03_system/js-apis-cardemulation.md` - `hasHceCapability`, `HceService`, `start`, `on('hceCmd')`, `transmit`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-cardemulation
- `documentation/harmonyos-references/03_system/js-apis-nfctag.md` - `tag.on('readerMode')`, `TagInfo`, `getNfcA`, `getNdef`, `getIsoDep`, `ndef.makeUriRecord`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-nfctag
- `documentation/harmonyos-references/02_application-framework/js-apis-inner-application-uiabilitycontext.md` - `openLink` and `appLinkingOnly`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-inner-application-uiabilitycontext
- `documentation/harmonyos-references/08_common-capabilities/js-apis-syscap.md` - `canIUse` and the NFC syscap string
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-syscap
- `documentation/harmonyos-references/02_application-framework/ts-container-tabcontent.md` - `onWillShow` / `onWillHide` ordering
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-tabcontent
- `documentation/harmonyos-guides/04_system/nfc-hce-guide.md` - the HCE card-side flow and AID registration
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/nfc-hce-guide
- `documentation/harmonyos-guides/04_system/nfc-tag-access-guide.md` - foreground reader mode and technology selection
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/nfc-tag-access-guide
- `documentation/harmonyos-guides/04_system/permissions-for-all.md` - `ohos.permission.NFC_TAG`, `ohos.permission.NFC_CARD_EMULATION`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all
