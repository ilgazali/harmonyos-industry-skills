---
id: UTIL-21
title: NFC tag read and write - one readerMode registration, two callbacks, NDEF records parsed by TNF
industry: 15_utilities
doc: huawei_industry_tree/15_utilities/docs/21_nfc_read_write.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/nfc_read_write-0000002299096544
sample: huawei_industry_tree/15_utilities/downloads/NFCReadWrite.zip
kits: ["@kit.AbilityKit", "@kit.ArkTS", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.ConnectivityKit", "@kit.LocalizationKit", "@kit.PerformanceAnalysisKit"]
apis: [Want, buffer, bundleManager, hilog, nfcController, resourceManager, tag, util, window]
permissions: [ohos.permission.NFC_TAG]
min_api: 20
modules: [entry (entry)]
findings: [HW-15-0053, HW-15-0054, HW-15-0101, HW-15-0102]
status: verified-with-fixes
---

## When to use

**Load this card when the app has to talk to a physical NFC tag** - read what
is on it, or program it. The pattern here is foreground dispatch: while the
app is on screen it claims every tag the device discovers, instead of letting
the system route the tag to whatever app declared a matching intent.

The mechanism is one registration API, `tag.on('readerMode', ...)`, with the
mode of the app decided entirely by **which callback you pass in**. Read mode
passes `readerModeCb`; write mode passes `writeModeCb`. Nothing else changes -
same element name, same tech list. That is the transferable idea: a
"foreground reader" is a subscription, and the behaviour on discovery is the
subscription's payload.

It generalises past this demo to inventory scanning, access badges, transit
card inspection, product-authenticity checks - anything where a user taps a
tag against the phone with your app already open. The complementary shape is
HCE (the device pretending to *be* the card); see `UTIL-42`, which shares this
sample's `NdefTagModel` and therefore also shares `HW-15-0053`.

## Feature checklist

- A two-tab home page: 读卡 (read card) and 写卡 (write card).
- On entering the read tab, the app registers foreground dispatch for
  `NFC_A`, `NDEF` and `MIFARE_ULTRALIGHT`.
- Touching a tag fills in UID (colon-separated hex), the technology list by
  name, SAK, ATQA, whether the tag is NDEF-writable and whether it can be made
  read-only.
- Every NDEF record on the tag is listed with a type label (网址 URL / 文本
  text / 启动应用 launch-app) and its decoded content.
- The write tab offers three record kinds - URI, text, external - each opening
  a sheet with a single text field.
- 添加记录 appends the record to a local list shown at the bottom of the page;
  写入 arms write mode and programs the next tag touched.
- A toast reports write success or failure, and write mode disarms itself
  afterwards.
- Leaving the read tab, closing a sheet, or leaving the page unregisters the
  dispatch.

## Architecture

One `entry` module. The NFC concern is fully separated from the UI: a
singleton util owns the subscription, two models own the per-technology
decoding.

```
entry/src/main/ets
├── constants/CommonConstant.ets   background colour + the two percent literals
├── constants/Constant.ets         tab-bar and title typography
├── entryability/EntryAbility.ets  canIUse(NFC.Core) guard, NfcUtil.init(want), avoid areas -> AppStorage
├── model/NdefTagModel.ets         NDEF: read records, decode by TNF, build records, writeNdef
├── model/NfcATagModel.ets         NFC-A: SAK and ATQA as hex strings
├── pages/HomePage.ets             @Entry, Navigation + two tabs, owns dispatch on/off
├── pages/ReadCardPage.ets         read UI, renders straight off NfcUtil state
├── pages/WriteCardPage.ets        write UI, three sheets, the record list, showToast()
└── utils/NfcUtil.ets              the singleton: tag.on/off, readerModeCb, writeModeCb
    utils/UriPrefixUtil.ets        the 0x00-0x23 NDEF URI prefix table
```

The documented tree matches the zip exactly.

**The design decision worth copying** is that `NfcUtil` is an `@Observed`
singleton and both pages hold it in `@State`:

```typescript
@State nfcUtil: NfcUtil = NfcUtil.getInstance();
```

A tag arrives on a system callback, not inside a component's event handler, so
there is no component to write into. Making the util the observed object means
`readerModeCb` can assign `isIdentified`, `uidHexString`, `nfcA` and `ndef` as
plain fields and both pages re-render. The callback stays free of any UI
knowledge, and the same instance serves the read tab and the write tab - which
matters, because the write tab needs `records` to already be set before the
tag is touched.

The decision **worth avoiding** is `readerModeCb` returning early when
`getNfcA` yields null: a plain NDEF tag with no NFC-A layer is rejected before
its records are ever read, even though the whole read UI below the header is
driven by `ndef`. The tech list requested in `tag.on` is a union; the callback
treats it as a conjunction.

## Implementation steps

1. **Declare `ohos.permission.NFC_TAG` and the TAG_FOUND action.** The
   permission alone is not enough - the ability's `skills` must carry
   `"ohos.nfc.tag.action.TAG_FOUND"` alongside `action.system.home`, or the
   system never dispatches a tag to this ability.
2. **Guard on syscap in `onCreate`**: `canIUse('SystemCapability.Communication.NFC.Core')`
   before touching anything in the tag module, and return if it is absent.
3. **Build the `ElementName` from the launch `want`** - `bundleName`,
   `abilityName`, `moduleName` - and keep it; `tag.on` and `tag.off` both need
   the same value.
4. **Register with a tech list, dispatch by callback.** `TECH_LIST` is
   `[tag.NFC_A, tag.NDEF, tag.MIFARE_ULTRALIGHT]`; read and write differ only
   in the function handed to `tag.on`.
5. **Unregister on every exit path**: `onWillHide` of the read tab,
   `onWillDisappear` of a write sheet, `aboutToDisappear` of the page, and at
   the tail of `writeModeCb` once the write has been reported.
6. **Connect, operate, then reset the connection** (`HW-15-0054`). The tag
   session contract is three-part; the sample stops at two, so the same tag
   cannot be re-detected without lifting it away and back.
7. **Decode records by TNF first, RTD type second.** `TNF_WELL_KNOWN` splits
   into `'U'` and `'T'`; `TNF_EXT_APP` is the launch-app record.
8. **Decode UTF-16 text records with a real encoding** (`HW-15-0053`).
   `'utf-16'` is not a `BufferEncoding` value, and NDEF UTF-16 is big-endian
   with a BOM, so `'utf16le'` alone is also wrong.
9. **Default an unknown URI prefix code to the empty string** - the map covers
   0x00-0x23 only, and a miss otherwise concatenates `undefined` into the URL
   (`HW-15-0053`; the corruption is visible in `UTIL-42`'s copy).
10. **Build write records through the `tag.ndef` factories** -
    `makeUriRecord`, `makeTextRecord`, `makeExternalRecord` - never by hand;
    they own the prefix byte and the language-code header.

## Verified snippets

All snippets are from `NFCReadWrite.zip`. Corrected forms are marked.

**One registration, two modes — `entry/src/main/ets/utils/NfcUtil.ets`** (as shipped)

```typescript
import { nfcController, tag } from '@kit.ConnectivityKit';
import { bundleManager } from '@kit.AbilityKit';
import { AsyncCallback } from '@kit.BasicServicesKit';

const TECH_LIST: number[] = [tag.NFC_A, tag.NDEF, tag.MIFARE_ULTRALIGHT];

@Observed
export class NfcUtil {
  private static instance: NfcUtil;
  private nfcTagElementName: bundleManager.ElementName | undefined = undefined;
  private foregroundRegister: boolean | undefined = undefined;

  public init(want: Want): void {
    this.nfcTagElementName = {
      bundleName: want.bundleName ?? '',
      abilityName: want.abilityName ?? '',
      moduleName: want.moduleName,
    };
  }

  public isNfcOpen() {
    return nfcController.isNfcOpen();
  }

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
}
```

**Three details carry the design.** The `ElementName` comes from the launch
`want` rather than being constructed from constants, so the registration
survives a rename of the module. `foregroundRegister` makes `nfcTagOff`
idempotent - and it needs to be, because four different places call it
(`onWillHide`, the sheet's `onWillDisappear`, `aboutToDisappear` and the tail
of `writeModeCb`). And `tag.on` is wrapped in `try/catch` rather than checked:
registration throws rather than returning an error when the device has no NFC
or the permission was denied.

`isNfcOpen()` is a separate `nfcController` call - NFC being *supported* is a
syscap question answered in `EntryAbility`, NFC being *switched on* is a
runtime question answered here, and both pages ask it before registering.

**Opening the tag session — same file** (corrected, see `HW-15-0054`)

```typescript
public async readerModeCb(error: BusinessError, tagInfo: tag.TagInfo) {
  if (!error) {
    if (tagInfo === null || tagInfo === undefined) { return; }
    if (tagInfo.uid === null || tagInfo.uid === undefined) { return; }
    if (tagInfo.technology === null || tagInfo.technology === undefined ||
      tagInfo.technology.length === 0) { return; }

    NfcUtil.getInstance().isIdentified = true;
    NfcUtil.getInstance().uidHexString = getTagInfoUid(tagInfo);
    NfcUtil.getInstance().techListString = getTagInfoTech(tagInfo);

    let ndefTag: tag.NdefTag | null = null;
    for (let i = 0; i < tagInfo.technology.length; i++) {
      if (tagInfo.technology[i] === tag.NDEF) {
        try {
          ndefTag = tag.getNdef(tagInfo);
        } catch (error) {
          hilog.error(0x0000, 'testTag', 'getNdef error = %{public}s', JSON.stringify(error));
          return;
        }
      }
    }

    if (ndefTag !== null) {
      try {
        ndefTag.connect();
      } catch (error) {
        hilog.error(0x0000, 'testTag', 'connect error = %{public}s', JSON.stringify(error));
        return;
      }
      if (!ndefTag.isConnected()) { return; }
      try {
        NfcUtil.getInstance().ndef = new NdefTagModel(ndefTag);   // reads the records
      } finally {
        ndefTag.resetConnection();   // FIX: absent in the sample - the session is never released
      }
    }
  }
}
```

**`tag.getNdef(tagInfo)` does not open anything** - it is a typed view over the
`TagInfo` the system handed you. The session is opened by `connect()`,
confirmed by `isConnected()`, and released by `resetConnection()`. The sample
does the first two and never the third, on all three call sites
(`readerModeCb` twice, `writeModeCb` once). The reference is explicit that
`resetConnection` "resets the connection to this tag"; leaving it out keeps the
channel bound and blocks re-detection of the same tag, which is exactly what a
read demo invites the user to do.

Note the order in the read path: `new NdefTagModel(ndefTag)` calls
`getNdefMessage()` in its own constructor, so the records must be pulled
*before* the reset - hence `try/finally` rather than a reset appended after the
assignment.

**Decoding a text record — `entry/src/main/ets/model/NdefTagModel.ets`** (corrected, see `HW-15-0053`)

```typescript
static getUriContent(payload: number[]) {
  // 解析前缀协议
  let prefix = UriPrefixUtil.URI_PREFIX_MAP.get(payload[0]) ?? '';   // FIX: unknown code yielded undefined
  // 解析内容
  let content = payload.slice(1, payload.length + 1);
  return prefix + buffer.from(content).toString('utf-8');
}

static getTextContent(payload: number[]) {
  const isUtf16 = (payload[0] & 0x80) !== 0;     // status byte, bit 7: 0 = UTF-8, 1 = UTF-16
  const localeLength = payload[0] & 0x3f;        // bits 0-5: length of the IANA language code
  let body = payload.slice(localeLength + 1, payload.length);

  if (!isUtf16) {
    return buffer.from(body).toString('utf-8');
  }
  // FIX: the sample passes 'utf-16', which is not a BufferEncoding value.
  // NDEF UTF-16 is big-endian and may carry a BOM; swap to LE, drop the BOM, then decode.
  if (body[0] === 0xFE && body[1] === 0xFF) {
    body = body.slice(2);
  }
  const swapped: number[] = [];
  for (let i = 0; i + 1 < body.length; i += 2) {
    swapped.push(body[i + 1], body[i]);
  }
  return buffer.from(swapped).toString('utf16le');
}
```

**The first payload byte of an RTD-T record is a status byte, not content.**
Bit 7 selects the encoding, bits 0-5 give the length of the language code that
follows, so the text begins at `localeLength + 1`. The sample gets that part
right and then hands `'utf-16'` to `buffer.toString()`, which accepts
`'ascii' | 'utf8' | 'utf-8' | 'utf16le' | 'ucs2' | 'ucs-2' | 'base64' |
'base64url' | 'latin1' | 'binary' | 'hex'` and throws 401 for anything else.
So the entire UTF-16 branch is dead code that raises on the first non-ASCII
tag written by another platform.

`'utf16le'` on its own is still not the answer: NDEF specifies UTF-16 **big
endian**, optionally with a BOM. The corrected form strips the BOM and swaps
byte pairs before decoding.

The URI defaulting matters for the same class of reason: `URI_PREFIX_MAP`
covers prefix codes `0x00` through `0x23`, and a tag written with a newer or
malformed code makes `HashMap.get` return `undefined`, which string
concatenation renders as the literal text `undefined` in front of the URL.

**Arming write mode — `entry/src/main/ets/pages/WriteCardPage.ets`** (as shipped)

```typescript
Button($r('app.string.write_input_button_write'), { stateEffect: false })
  .onClick(() => {
    if (getRecord !== undefined) {
      let record = getRecord(this.inputContent);
      if (record) {
        this.nfcUtil.records = [record];
        this.nfcUtil.nfcTagOn(this.nfcUtil.writeModeCb);   // same API, other callback
        showToast($r('app.string.write_input_button_write_toast'), 3000, this.uiContext);
      }
    }
  })
```

```typescript
// NfcUtil.writeModeCb, tail
NfcUtil.getInstance().ndef = new NdefTagModel(ndefTag);
let res = await NfcUtil.getInstance().ndef?.writeNdefToTag(NfcUtil.getInstance().records);
let context = NfcUtil.getInstance().context;
if (context) {
  showToast(res ? $r('app.string.nfc_write_success') : $r('app.string.nfc_write_fail'), 2000, context);
}
// ...
NfcUtil.getInstance().nfcTagOff();     // write mode is one-shot
```

**Write mode is armed, fired, and disarmed.** The button does not write - it
stages `records` on the singleton and swaps the dispatch callback, then the
toast tells the user to present a tag. The actual `writeNdef` happens inside
`writeModeCb`, which finishes by calling `nfcTagOff()` unconditionally, so a
second tap of the same tag does not silently rewrite it.

`showToast` is exported from the page and imported by `NfcUtil` - which is why
`NfcUtil.context` exists at all: the singleton is handed the page's
`UIContext` in `HomePage.aboutToAppear` so a system callback can raise UI. That
is the pragmatic half of the singleton design and the part to keep an eye on;
a longer-lived app should publish an event instead of holding a UI context on
a static.

## Permissions & config

```json5
"requestPermissions": [
  {
    // 添加nfc标签操作的权限
    "name": "ohos.permission.NFC_TAG",
    "reason": "$string:NFC_reason",
  }
],
"abilities": [
  {
    "skills": [
      {
        "entities": ["entity.system.home"],
        "actions": [
          "action.system.home",
          // actions须包含"ohos.nfc.tag.action.TAG_FOUND"
          "ohos.nfc.tag.action.TAG_FOUND"
        ]
      }
    ]
  }
]
```

- `ohos.permission.NFC_TAG` is `system_grant` at normal level - no runtime
  dialog, but `reason` is still declared here.
- `TAG_FOUND` in `skills.actions` is the non-obvious half. Foreground dispatch
  registers against an `ElementName`; the system will only hand a tag to an
  ability that advertises the action.
- `deviceTypes` is `["phone"]` only, which is honest: the tag APIs are
  documented for phone and wearable.
- No `tag-tech` metadata filter is declared, because the tech list is passed
  at registration time instead.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`,
  so the constraint and the sample agree.
- The document warns that importing the `tag` module may raise an editor error
  because the capability exceeds the project's default device capability set;
  the fix is a custom syscap declaration, not a code change.
- 是否可写 / 可为只读 in the read UI and the whole write tab need a tag with
  an NDEF layer. `MIFARE_ULTRALIGHT` tags that were never NDEF-formatted show
  only the UID and tech list.
- The read path additionally requires an NFC-A layer, because `readerModeCb`
  returns early when `getNfcA` is null - a plain NDEF tag never reaches record
  parsing.
- `writeNdefToTag` returns `false` rather than throwing on a locked or
  undersized tag; the UI distinguishes only success from failure, with no
  reason.
- The device must have NFC switched on. `isNfcOpen()` is checked but the app
  only logs and returns - there is no prompt to open settings.

## Pitfalls

- **`HW-15-0053`** (B/medium, probable): the UTF-16 branch of `getTextContent`
  passes `'utf-16'` to `buffer.toString()`, which is not a supported
  `BufferEncoding` and throws 401; even `'utf16le'` would mis-decode, because
  NDEF UTF-16 is big-endian with a BOM. Systematic - `UTIL-42`'s `HceLinkApp`
  ships the same `NdefTagModel`, and there the unknown-URI-prefix path also
  concatenates the literal `undefined` into the parsed URL. Fix: decode UTF-16
  properly (strip BOM, swap to LE) and default the prefix lookup to `''`.
- **`HW-15-0054`** (E/low, confirmed): tag sessions opened with `connect()`
  are never released - `NfcUtil.ets:146`, `:163` and `:224` all connect,
  operate, and stop. `nfcTagOff` only unregisters the `readerMode` listener.
  The tag-kit contract is connect → operate → `resetConnection`, and an open
  session blocks re-detection of the same tag. Fix: add
  `resetConnection()` after each operation, in a `finally`.
- Not a filed finding but worth knowing: `readerModeCb` aborts when the tag
  has no NFC-A layer, before any NDEF record is read, although the tech list
  registered includes NDEF on its own.

## References

- `documentation/harmonyos-references/03_system/js-apis-nfctag.md` -
  `tag.on('readerMode')`, `TnfType`, `tag.ndef.makeUriRecord` /
  `makeTextRecord` / `makeExternalRecord` / `createNdefMessage`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-nfctag
- `documentation/harmonyos-references/03_system/js-apis-nfctech.md` -
  `NdefTag.getNdefMessage`, `writeNdef`, `isNdefWritable`, `canSetReadOnly`;
  `NfcATag.getSak` / `getAtqa`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-nfctech
- `documentation/harmonyos-references/03_system/js-apis-tagsession.md` -
  `connect`, `isConnected` and `resetConnection` (the contract behind `HW-15-0054`)
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-tagsession
- `documentation/harmonyos-references/02_application-framework/js-apis-buffer.md` -
  the `BufferEncoding` union that `'utf-16'` is not a member of
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-buffer
- `documentation/harmonyos-guides/04_system/nfc-tag-access-guide.md` - the
  foreground-dispatch flow end to end
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/nfc-tag-access-guide
- `documentation/harmonyos-guides/04_system/permissions-for-all.md` -
  `ohos.permission.NFC_TAG`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all
- `documentation/harmonyos-guides/01_getting-started/module-configuration-file.md` -
  the `skills` tag and `ohos.nfc.tag.action.TAG_FOUND`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/module-configuration-file
- `documentation/harmonyos-references/01_api-reference-overview/syscap.md` -
  declaring a custom syscap when the editor rejects the `tag` import
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/syscap
- `UTIL-42` - HCE, the other side of the same conversation, and the second
  home of this `NdefTagModel`
