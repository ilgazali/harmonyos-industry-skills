---
id: UTIL-30
title: HMAC compute and verify - a symmetric key from user text, SHA256 MAC, hex on screen
industry: 15_utilities
doc: huawei_industry_tree/15_utilities/docs/30_calculation_and_verification_of_hmac.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/calculation_and_verification_of_hmac-0000002431485178
sample: huawei_industry_tree/15_utilities/downloads/DataEncryptionBasedOnHAMC.zip
kits: ["@kit.CryptoArchitectureKit", "@kit.ArkTS", "@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: ["cryptoFramework.createSymKeyGenerator", "SymKeyGenerator.convertKey", "cryptoFramework.createMac", "Mac.init", "Mac.update", "Mac.doFinal", "buffer.from", TextInput, onWillInsert, "@Link", "@StorageProp", showToast]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-15-0068, HW-15-0101]
status: verified-with-fixes
---

## When to use

**Load this card when two parties share a secret and one of them needs to
prove a message was not tampered with.** HMAC is the answer whenever you have a
shared key and want integrity plus origin - signed API requests, a webhook
receiver, a locally stored blob you want to detect edits to, a QR payload that
must not be forged offline. It is not encryption: the message stays readable,
what you gain is a tag that only a holder of the key could have produced.

The pattern here is the minimum viable shape of it: turn a key string into a
`SymKey` with `createSymKeyGenerator('HMAC|SHA256')` + `convertKey`, feed the
message through `Mac.init` / `update` / `doFinal`, and render the resulting
bytes as hex. Verification is the same computation run again and compared.

**The part that generalises worst is the key handling**, and it is exactly
where the sample is wrong. A MAC key is a byte string, not a character string.
The sample gates its buttons on `secretKey.length === 32` - 32 UTF-16 code
units - while `HMAC|SHA256` wants 32 bytes, and the two agree only for pure
ASCII (`HW-15-0068`). Any card you build on this one must count bytes.

## Feature checklist

- A two-tab switch at the top: 计算 (compute) and 校验 (verify) mode, styled
  as a segmented control rather than a `Tabs` container.
- Each mode shows one card with two inputs: the message and the 密钥 (secret
  key), both marked mandatory with a red asterisk.
- Leading spaces are rejected as you type in the message field.
- The action button stays disabled until both the message and a key of the
  required length are present.
- Compute mode shows the MAC below the button as lowercase hex, copyable
  in-app, capped at two lines.
- Verify mode recomputes the MAC over the entered message and key and raises a
  success or failure toast.
- Switching modes keeps the key (it is shared state) but swaps the message
  field.

## Architecture

One `entry` module, seven files, and a clean separation: nothing in `utils`
knows about the UI, and nothing in the UI knows what a `DataBlob` is.

```
entry/src/main/ets
├── components/InputCard.ets      the message + key card, two @Link bindings
├── constants/CommonConstants.ets all sizes, paddings, and SECRET_KEY_LENGTH
├── entryability/EntryAbility.ets full-screen window, avoid areas -> AppStorage
├── entrybackupability/EntryBackupAbility.ets
├── model/HMACMode.ets            a two-value enum, EncryptionMode/VerificationMode
├── pages/MainPage.ets            @Entry, the mode switch and both flows
└── utils/HMACUtils.ets           genSymKeyByData / computeHmac / verifyHmac / hex
```

The documented tree matches the zip exactly, file for file and comment for
comment.

**The design decision worth copying** is `HMACUtils.ets`: four free functions,
no class, no state, no imports beyond `cryptoFramework` and `buffer`. The
whole crypto surface the page can see is

```typescript
computeHmac(data: string, secretKey: string): Promise<Uint8Array>
verifyHmac(data: string, secretKey: string, receivedMac: Uint8Array): Promise<boolean>
uint8ArrayToHex(buffer: Uint8Array): string
```

Strings in, bytes out, hex for display. `genSymKeyByData` is deliberately not
exported - the key object never escapes the module, so no caller can hold a
`SymKey` past the call that needed it. Swapping SHA256 for SHA512 is a
two-line change inside this file and nothing else moves.

**What the page adds to it is less clean.** `receivedMac` is a plain field on
`MainPage` (not `@State`), written only by the compute button. Verify mode
compares a freshly computed MAC against whatever that field happens to hold,
so verification is a self-check within one session - compute, then verify -
rather than a check of a MAC that arrived from somewhere. There is no input
for a received MAC anywhere in the UI. Treat the verify tab as a demonstration
of `verifyHmac`, not as a receiving end.

## Implementation steps

1. **Convert the key, do not generate it.** `createSymKeyGenerator('HMAC|SHA256')`
   plus `convertKey(blob)` turns caller-supplied bytes into a `SymKey`;
   `generateSymKey` would make a random one, which is not what a shared secret
   is.
2. **Validate the key by byte length, not character count** (`HW-15-0068`).
   `'字'.length` is 1 and its UTF-8 encoding is 3 bytes; a 32-character
   Chinese key is 96 bytes and `convertKey` rejects it.
3. **Wrap the call in try/catch** (`HW-15-0068`). `convertKey` rejects
   asynchronously, and an `async` `onClick` with no catch produces an unhandled
   rejection and a silent no-op button.
4. **Set the "we have a result" flag after the await, not before**
   (`HW-15-0068`). The sample sets `isEncrypted = true` on the first line of
   the handler, so on failure the result block renders with an empty MAC in it.
5. **Feed the message through `update` before `doFinal`.** `update` may be
   called repeatedly for a streamed message; `doFinal` closes the MAC and
   returns it.
6. **Render as hex with `padStart(2, '0')`.** Without the pad, any byte below
   0x10 loses a digit and the whole tag shifts.
7. **Compare MACs as bytes.** The sample's `toString()` comparison is
   readable but variable-time; for a MAC checked against attacker-supplied
   input, compare with a constant-time loop over the full length.

## Verified snippets

All snippets are from `DataEncryptionBasedOnHAMC.zip`. Corrected forms are
marked.

**The crypto module — `entry/src/main/ets/utils/HMACUtils.ets`** (as shipped)

```typescript
import { cryptoFramework } from '@kit.CryptoArchitectureKit';
import { buffer } from '@kit.ArkTS';

// 生成对称密钥
async function genSymKeyByData(symKeyData: Uint8Array): Promise<cryptoFramework.SymKey> {
  let symKeyBlob: cryptoFramework.DataBlob = { data: symKeyData };
  let aesGenerator = cryptoFramework.createSymKeyGenerator('HMAC|SHA256');
  let symKey = await aesGenerator.convertKey(symKeyBlob);
  return symKey;
}

// 计算HMAC
export async function computeHmac(data: string, secretKey: string): Promise<Uint8Array> {
  let keyData = new Uint8Array(buffer.from(secretKey, 'utf-8').buffer);
  let key = await genSymKeyByData(keyData);
  let macAlgName = 'SHA256'; // 摘要算法名，根据具体采用摘要算法设置
  let mac = cryptoFramework.createMac(macAlgName);
  await mac.init(key);
  let message = data; // 待进行HMAC的数据
  await mac.update({ data: new Uint8Array(buffer.from(message, 'utf-8').buffer) });
  let macResult = await mac.doFinal();
  return macResult.data;
}
```

**Two different algorithm strings, and the difference matters.** The *key
generator* is created with `'HMAC|SHA256'` - the full specifier, algorithm plus
digest, which is what fixes the required key size at 32 bytes. The *Mac object*
is created with `'SHA256'` alone, because `createMac` takes only the digest
name; the "HMAC" part is implied by the API. Passing `'HMAC|SHA256'` to
`createMac` fails, and passing `'SHA256'` to `createSymKeyGenerator` fails.
They look like a typo for each other and are both correct.

`init` / `update` / `doFinal` is the standard three-step: `init` binds the key,
`update` accepts the message and may be called many times for a large or
streamed payload, `doFinal` closes the computation and returns the tag. A `Mac`
object is not reusable after `doFinal` - which is why `computeHmac` builds a
fresh one per call rather than caching it.

**The compute button — `entry/src/main/ets/pages/MainPage.ets`** (corrected, see `HW-15-0068`)

```typescript
import { buffer } from '@kit.ArkTS';            // FIX: not imported by the page today

Button($r('app.string.EncryptionMode'))
  .width(CommonConstants.FULL_WIDTH)
  .height(CommonConstants.BUTTON_HEIGHT)
  // FIX: shipped code is `this.secretKey.length === CommonConstants.SECRET_KEY_LENGTH`,
  // which counts characters; HMAC|SHA256 needs 32 BYTES.
  .enabled(this.unencryptedData !== '' &&
    buffer.from(this.secretKey, 'utf-8').length === CommonConstants.SECRET_KEY_LENGTH)
  .margin({ top: CommonConstants.MARGIN_SIXTEEN })
  .onClick(async () => {
    try {                                        // FIX: no try/catch in the sample
      let unit8Array = await computeHmac(this.unencryptedData, this.secretKey);
      this.receivedMac = unit8Array;
      this.encryptedData = uint8ArrayToHex(unit8Array);
      this.isEncrypted = true;                   // FIX: shipped code sets this first
    } catch (err) {
      this.isEncrypted = false;
      this.getUIContext().getPromptAction().showToast({ message: `${err.code}` });
    }
  });
```

**Three small changes, one bug each.** Counting bytes closes the gap between
what the button promises and what `convertKey` accepts: an ASCII key of 32
characters is 32 bytes and works, a 32-character key with any non-ASCII
character in it is 33 to 96 bytes and is rejected - by an enabled button, with
no message. The try/catch turns that rejection into feedback instead of an
unhandled promise. And moving `isEncrypted = true` below the await stops the
result panel from appearing with nothing in it: the flag means "there is a MAC
to show", so it must be set once there is one.

`CommonConstants.SECRET_KEY_LENGTH = 32` stays as it is - the number is right,
it was only being applied to the wrong quantity.

**The shared input card — `entry/src/main/ets/components/InputCard.ets`** (as shipped)

```typescript
@Component
export struct InputCard {
  @Prop label: Resource;
  @Link inputValue: string;
  @Link secretKey: string;

  build() {
    Column() {
      // ... 必填 (mandatory) marker + the label ...
      TextInput({ placeholder: $r('app.string.please_input'), text: this.inputValue })
        .showUnderline(false)
        .backgroundColor(Color.Transparent)
        .width(CommonConstants.FULL_WIDTH)
        .padding({ left: CommonConstants.PADDING_ZERO })
        .onChange((value: string) => {
          this.inputValue = value;
        })
        .onWillInsert((info: InsertValue) => {
          if (info.insertValue.startsWith(' ')) {
            return false;                        // reject a leading space before it lands
          } else {
            return true;
          }
        });
      Divider().strokeWidth(CommonConstants.STROKE_WIDTH)
      // ... the same pair again for 密钥 (secret key), bound to this.secretKey ...
    }
  }
}
```

**`@Prop` for the label, `@Link` for the values.** The label is a one-way
`Resource` that changes only when the parent swaps modes; the two text values
have to travel back up, because the parent's button enablement and the crypto
call both read them. That is the whole reason this card can be reused by both
modes with a different label and a different message binding while sharing the
key binding - `MainPage` passes `secretKey: this.secretKey` in both branches,
so typing the key once serves compute and verify.

`onWillInsert` is the input filter worth remembering: it runs *before* the
character enters the field and a `false` return drops it, so the state never
sees the bad value. Doing the same job in `onChange` means writing the
rejected text and then rewriting the field, which fights the cursor.

**Verification — `entry/src/main/ets/utils/HMACUtils.ets`** (as shipped)

```typescript
// 验证HMAC
export async function verifyHmac(data: string, secretKey: string,
                                 receivedMac: Uint8Array): Promise<boolean> {
  let computedMac = await computeHmac(data, secretKey);
  return computedMac.toString() === receivedMac.toString();
}

// Uint8Array转Hex
export function uint8ArrayToHex(buffer: Uint8Array): string {
  return Array.from(buffer)
    .map(byte => byte.toString(16).padStart(2, '0'))
    .join('');
}
```

**Verification is recomputation.** There is no "verify" primitive in the MAC
API and there does not need to be one: you hold the same key, so you produce
the tag again and compare. This is the structural difference from a signature,
where the verifier holds a *different* (public) key and calls a real verify
operation.

`Uint8Array.toString()` yields the comma-joined decimal bytes, so the
comparison is correct but returns as soon as the strings differ. For a MAC
checked against a value an attacker controls, that timing is a real side
channel; compare byte by byte over the full length with an accumulating XOR
instead.

`padStart(2, '0')` in the hex helper is not cosmetic. Without it `0x0a` renders
as `a`, the string gets shorter, and every downstream byte boundary is off.

## Permissions & config

**None.** `module.json5` declares no `requestPermissions`; HMAC is a pure
in-process computation.

`deviceTypes` is `["phone", "tablet", "2in1"]`, and an `EntryBackupAbility` is
declared with `$profile:backup_config` - standard boilerplate, unrelated to the
feature.

Layout constants all live in `CommonConstants.ets` as named numbers
(`PADDING_TWELVE`, `MARGIN_SIXTEEN`, `CARD_BORDER_RADIUS`). `SECRET_KEY_LENGTH`
is filed among them, which is part of why it reads like a UI constant and got
applied like one.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **The key length is fixed at 32 bytes by `HMAC|SHA256`.** Other digests want
  other sizes; if you make the digest configurable, the validation constant has
  to move with it.
- The key is typed into a plain `TextInput` with no masking and no `.type()`
  set, so it is visible on screen and goes through the normal IME. A real
  secret entry field would use `InputType.Password`.
- The MAC display is `maxLines(2)` with a hex SHA256 tag of 64 characters, so
  it fits on a phone but will clip on narrow windows. `copyOption(CopyOptions.InApp)`
  still copies the full value.
- Verify mode has no field for a received MAC. It can only re-check the tag
  produced by the compute tab in the same session (see Architecture).
- `receivedMac` is a plain class field, not `@State`, which is correct here -
  nothing renders it - but means it survives a mode switch and silently
  resets on page recreation.

## Pitfalls

- **`HW-15-0068`** (B/medium, probable): the key is validated as 32
  *characters* (`MainPage.ets:99` and `:145`) while `'HMAC|SHA256'` requires 32
  *bytes*, so a legal-looking non-ASCII key passes the check, `convertKey`
  rejects it, and the `async onClick` has no try/catch - the button does
  nothing at all. Compounding it, `isEncrypted = true` is set before the await
  (`:101-106`), so the result panel opens even when the computation failed.
  Fix: validate `buffer.from(key, 'utf-8').length`, wrap the handler in
  try/catch with a toast, and set the flag only after the MAC exists.

## References

- `documentation/harmonyos-references/03_system/js-apis-cryptoframework.md` -
  `createSymKeyGenerator`, `convertKey`, `createMac`, `Mac.init/update/doFinal`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-cryptoframework
- `documentation/harmonyos-guides/04_system/crypto-compute-hmac.md` - the
  compute flow and the key-size requirement per digest
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/crypto-compute-hmac
- `documentation/harmonyos-guides/04_system/crypto-compute-mac-overview.md` -
  what a MAC is for and when to prefer it over a signature
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/crypto-compute-mac-overview
- `documentation/harmonyos-references/02_application-framework/js-apis-buffer.md` -
  `buffer.from` and byte length
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-buffer
- `UTIL-29` - the sibling crypto card; RSA over the same framework, and the
  same `buffer` conversion idiom done right and wrong
