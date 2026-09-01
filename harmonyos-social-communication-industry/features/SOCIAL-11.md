---
id: SOCIAL-11
title: SM2 message encryption in a chat - encrypt on send, show the Base64 ciphertext and the decrypted text as two bubbles
industry: 14_social_communication
doc: huawei_industry_tree/14_social_communication/docs/11_encryp_chat.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/encryp_chat-0000002291169549
sample: huawei_industry_tree/14_social_communication/downloads/EncrypChat.zip
kits: ["@kit.AbilityKit", "@kit.ArkTS", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.CryptoArchitectureKit", "@kit.PerformanceAnalysisKit"]
apis: [buffer, cryptoFramework, hilog, util, window]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-14-0024, HW-14-0025, HW-14-0087]
status: verified-with-fixes
---

## When to use

Load this card when a chat has to **put ciphertext on the wire instead of
plaintext** and you need the ArkTS shape of an SM2 encrypt/decrypt round trip:
which generator string, which cipher string, what `DataBlob` is, and how the
binary result becomes something transportable.

The sample is a *demonstration*, not a protocol: it generates a fresh keypair
per message, encrypts with the public key and immediately decrypts with the
private key in the same function, then shows both results as two chat bubbles -
one labelled 加密消息 (encrypted message), one labelled 解密消息 (decrypted
message). Everything about the crypto calls transfers to a real design; nothing
about the key lifecycle does. In a real app the peer's public key arrives out of
band, the private key lives in the Universal Keystore, and SM2 wraps a symmetric
session key rather than the message body.

The part that generalises furthest is unglamorous: **the encoding boundary.**
Text becomes UTF-8 bytes, bytes become a `DataBlob`, ciphertext bytes become
Base64 for display or transport, and plaintext bytes must become text again
through a real decoder. Both findings on this card are failures at exactly that
boundary, and they are the failures every hand-rolled crypto layer makes.

## Feature checklist

- A chat screen with a title, a scrolling message list and a `RichEditor` input
  row.
- Typing enables the send button; sending clears the editor.
- Each send appends two bubbles: the Base64 ciphertext under a 加密消息 label,
  and the decrypted text under a 解密消息 label.
- The decrypted bubble reproduces the typed text exactly - including emoji and
  punctuation.
- The sender side alternates on each send, so the two bubble alignments and
  avatars are both exercised.
- The list scrolls to the bottom whenever a message is appended.
- The soft keyboard resizes the page rather than pushing it up.

## Architecture

One `entry` module, one page, one view, one model and one utility.

```
entry/src/main/ets
├── entryability/EntryAbility.ets    full screen, avoid areas -> AppStorage (raw px)
├── entrybackupability/EntryBackupAbility.ets
├── model/DataModel.ets              MsgContent { isSelf, isCrypt, content } and MsgTextImage
├── pages/Chat.ets                   @Entry: the bubble list, the @Provide'd state
├── utils/CipherUtil.ets             a hand-written Uint8Array -> string decoder
└── view/MessageSend.ets             the input row, and the whole crypto path
```

The documented 工程目录 matches the zip. (The tree-mismatch systematic
`HW-14-0001` covers four other social samples, not this one.)

State is shared with `@Provide`/`@Consume` rather than props: `Chat` provides
`data`, `isSelf`, `isHaveMsg` and `isComponentFocused`, and `MessageSend`
consumes all four. `MessageSend({})` is therefore constructed with no
parameters at all. For a two-component demo that is fine; past that,
`isComponentFocused` travelling upward through a provider so the page can drop
its bottom padding while the keyboard is open is the kind of coupling that gets
hard to trace.

The `RichEditorController` is a **module-level `const`**, exported from
`MessageSend.ets` and imported by `Chat.ets` so the list's `onClick` can call
`stopEditing()`. That is a singleton controller shared across the module - it
works because there is exactly one editor, and breaks the moment there are two.

**The design decision worth copying** is that `MsgContent` carries `isCrypt`
rather than two message arrays. One list renders both bubble kinds, the label
above the bubble is chosen by `visibility(item.isCrypt ? ... )`, and the
ciphertext and plaintext of one send stay adjacent and in order because they
are pushed back to back. **The decision worth avoiding** is that the crypto,
the encoding, the string surgery and the view-model push all live in one
`handlecCrypto` method on the component - which is why `HW-14-0025`'s
`slice(16, len - 4)` could ever look reasonable.

## Implementation steps

1. **Collect the editor contents from `getSpans()`** and join every text span -
   not just the first (`HW-14-0025`).
2. **Create the generator and the keypair**:
   `cryptoFramework.createAsyKeyGenerator('SM2_256').generateKeyPairSync()`.
   In a real app, import the peer's public key here instead.
3. **Turn the text into a `DataBlob`** with `buffer.from(content, 'utf-8')` -
   the crypto framework takes bytes, never strings.
4. **Encrypt with `createCipher('SM2_256|SM3')`**, `init(ENCRYPT_MODE, pubKey,
   null)`, then `doFinal`.
5. **Base64-encode the ciphertext for display or transport.** SM2 output is
   binary and cannot be put in a text field as-is.
6. **Decrypt with a second cipher object** initialised in `DECRYPT_MODE` with
   the private key. Do not reuse the encrypting cipher.
7. **Decode the plaintext bytes with `util.TextDecoder`**, not with a
   hand-written switch (`HW-14-0024`).
8. **Push the two bubbles** with `isCrypt` true then false, and let the page's
   `@Watch` scroll the list to the end.

## Verified snippets

All snippets are from `EncrypChat.zip`. Corrected forms are marked.

**The SM2 round trip — `entry/src/main/ets/view/MessageSend.ets`** (as shipped)

```typescript
import { cryptoFramework } from '@kit.CryptoArchitectureKit';
import { buffer, util } from '@kit.ArkTS';
import CipherUtil from '../utils/CipherUtil';

const base64Helper = new util.Base64Helper();

async handlecCrypto(content: string) {
  let generator = cryptoFramework.createAsyKeyGenerator('SM2_256');
  // 生成秘钥 - a fresh keypair for every single message
  let keyPair = generator.generateKeyPairSync();
  let plainText: cryptoFramework.DataBlob = { data: new Uint8Array(buffer.from(content, 'utf-8').buffer) };
  // 根据内容加密
  let encryptText = await this.encryptMessagePromise(keyPair.pubKey, plainText);
  let encryptData = base64Helper.encodeToStringSync(encryptText.data);
  // 根据秘文解密
  let decryptText = await this.decryptMessagePromise(keyPair.priKey, encryptText);
  let decryptData = CipherUtil.uint8ArrayToString(decryptText.data);
  // ...
}

// 加密消息
async encryptMessagePromise(publicKey: cryptoFramework.PubKey, plainText: cryptoFramework.DataBlob) {
  let cipher = cryptoFramework.createCipher('SM2_256|SM3');
  await cipher.init(cryptoFramework.CryptoMode.ENCRYPT_MODE, publicKey, null);
  let encryptData = await cipher.doFinal(plainText);
  return encryptData;
}

// 解密消息
async decryptMessagePromise(privateKey: cryptoFramework.PriKey, cipherText: cryptoFramework.DataBlob) {
  let decoder = cryptoFramework.createCipher('SM2_256|SM3');
  await decoder.init(cryptoFramework.CryptoMode.DECRYPT_MODE, privateKey, null);
  let decryptData = await decoder.doFinal(cipherText);
  return decryptData;
}
```

**Three strings carry the whole configuration.** `'SM2_256'` selects the curve
for the key generator; `'SM2_256|SM3'` selects the algorithm *and* the digest
for the cipher - SM2 encryption is defined in terms of a hash, and the two
halves of the string must agree between the encrypting and the decrypting
object. The third parameter of `init` is the algorithm parameter spec, `null`
for SM2 because there is no IV.

Two structural notes. Encryption and decryption use **separate `Cipher`
objects**: a cipher is stateful after `init`, so reusing one across modes is
not supported. And the keypair is generated inside the per-message handler, so
each message is encrypted under a key that exists for the length of one
function call - the demo can decrypt because it holds both halves, and nobody
else ever could. Real code creates or imports the keypair once and keeps the
private key in the Universal Keystore, never in a component field.

Note also that SM2 is asymmetric and slow, with an input-size limit tied to the
curve; a long message must be split, or - the normal design - encrypted with a
symmetric key that SM2 wraps.

**Bytes back to text — `entry/src/main/ets/utils/CipherUtil.ets`** (corrected, see `HW-14-0024`)

```typescript
import { util } from '@kit.ArkTS';

class CipherUtil {
  private decoder: util.TextDecoder = util.TextDecoder.create('utf-8');

  /**
   * uint8array转字符串
   */
  uint8ArrayToString(array: Uint8Array): string {
    // FIX: replaces a hand-written switch on (character >> 4) that handled
    // cases 0-7, 12-13 and 14 only - 4-byte sequences fell into `default` and
    // were silently dropped.
    return this.decoder.decodeToString(array);
  }
}

export default new CipherUtil();
```

The shipped decoder walks the array and branches on the top nibble of each lead
byte: `0-7` is ASCII, `12-13` is a two-byte sequence, `14` is a three-byte
sequence. Case `15` - lead bytes `0xF0`-`0xF4`, the four-byte sequences that
encode everything above U+FFFF - hits `default: break;` and the byte is thrown
away, along with its three continuation bytes on the next passes.

**That is every emoji.** Type `hello😀world`, and the decrypted bubble reads
`helloworld`: the round trip the sample exists to demonstrate visibly fails on
the single most common character class in a chat app. `util.TextDecoder`
handles the full range, and also handles the malformed input a hand-rolled loop
walks off the end of. There is no reason to write this function.

**Collecting the editor contents — `entry/src/main/ets/view/MessageSend.ets`** (corrected, see `HW-14-0025`)

```typescript
private async sendMessage() {
  let message: MsgTextImage[] = [];
  richController.getSpans().forEach(span => {
    message.push({
      content: (span as RichEditorTextSpanResult).value.toString()
    });
  });
  if (message.length > 0) {
    // FIX: the sample sends JSON.stringify(message[0]) - one span only, wrapped in JSON
    const text: string = message.map((m: MsgTextImage) => m.content as string).join('');
    this.handlecCrypto(text);
  }
  richController.deleteSpans();
}

async handlecCrypto(content: string) {
  // ... encrypt / decrypt as above ...
  let encryptData: string = base64Helper.encodeToStringSync(encryptText.data);
  let decryptData: string = CipherUtil.uint8ArrayToString(decryptText.data);

  // FIX: the sample double-stringifies both and then unwraps with fixed offsets:
  //   strEn = strEn.slice(1, strEn.length - 1);
  //   strDe = strDe.slice(16, strDe.length - 4);
  this.data.push({
    isCrypt: true,
    isSelf: this.isSelf,
    content: encryptData
  });
  this.data.push({
    isCrypt: false,
    isSelf: this.isSelf,
    content: decryptData
  });
}
```

**Two defects compound here, and the fix removes both by removing the JSON.**
The sample encrypts `JSON.stringify(message[0])`, so the plaintext is
`{"content":"hello"}` rather than `hello`; on the way out it stringifies again
and slices the wrapper off by counting characters - `slice(16, len - 4)` skips
exactly `"{\"content\":\"` and drops exactly `\"}"`. The offsets hold only while
the message contains no character that JSON escapes. Type a quotation mark, a
backslash, or a newline, and every subsequent character shifts: the bubble
renders truncated or with `\"` fragments visible.

A `RichEditor` also produces **one span per style run**. Typing
`hello😀world` yields several spans, and `message[0]` is the first of them - so
the sample encrypts `hello` and silently discards the rest. Joining the span
values and encrypting the text directly makes both problems disappear: nothing
is wrapped, so nothing has to be unwrapped, and `JSON.parse` is not needed
either.

**The bubble list — `entry/src/main/ets/pages/Chat.ets`** (as shipped)

```typescript
@Provide @Watch('onChangedData') data: MsgContent[] = [];
private listScroller: ListScroller = new ListScroller();

aboutToAppear(): void {
  this.getUIContext().setKeyboardAvoidMode(KeyboardAvoidMode.RESIZE); //虚拟键盘抬起时页面的避让模式
}

onChangedData() {
  this.listScroller.scrollEdge(Edge.End); //滚动至底部
}

// ... inside the List:
Text($r('app.string.encrypy_message'))
  .visibility(item.isCrypt ? Visibility.Visible : Visibility.None)
  .alignSelf(item.isSelf ? ItemAlign.Start : ItemAlign.End);
Text($r('app.string.decrypy_message'))
  .visibility(item.isCrypt ? Visibility.None : Visibility.Visible)
  .alignSelf(item.isSelf ? ItemAlign.Start : ItemAlign.End);
```

**`@Watch` on the provided array is what keeps the view at the bottom.** Every
`push` into `data` fires `onChangedData`, which drives the scroller to the end -
so the scroll rule lives next to the data instead of being repeated at each
call site.

`KeyboardAvoidMode.RESIZE` is the right choice for a chat: the page shrinks and
the list keeps its bottom edge above the keyboard, instead of the whole page
sliding up and the title disappearing. The page also drops its bottom
avoid-area padding while the editor holds focus (`isComponentFocused`), because
the navigation indicator is hidden behind the keyboard anyway.

`Visibility.None` (rather than `Hidden`) removes the unused label from layout,
so the two bubble kinds do not each reserve a blank line.

## Permissions & config

**None.** No `requestPermissions` block. The crypto runs entirely locally,
nothing is sent, and nothing is stored.

`deviceTypes` is `phone`, `tablet`, `2in1`. An `EntryBackupAbility` is declared
with `$profile:backup_config`, the standard template extension - it backs up
nothing meaningful here since there is no persisted state.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **This is not a messaging protocol.** The keypair is generated per message and
  discarded; the same component encrypts and decrypts. The document says as
  much - 示例中的密钥对为生成器生成，实际使用可以更换为使用者所提供的密钥对
  (the keypair here is generator-produced; real use should substitute the
  user's own). Treat the key lifecycle as unimplemented.
- SM2 is asymmetric: the plaintext size is bounded by the curve, and throughput
  is orders of magnitude below a symmetric cipher. Long messages need chunking,
  or the standard hybrid design (SM4 for the body, SM2 for the key).
- `generateKeyPairSync` is synchronous and runs on the UI thread on every send.
- The `RichEditorController` is a module-level singleton exported from
  `MessageSend.ets`; a second editor in the same module would share it.
- The `ForEach` key generator is `` `${JSON.stringify(item)}_${index}` ``, which
  is stable enough only because the index is included - two identical messages
  would otherwise collide.
- The message list is in-memory. Nothing survives a restart.

## Pitfalls

- **`HW-14-0024`** (B/medium, confirmed): the hand-written UTF-8 decoder in
  `CipherUtil.uint8ArrayToString` handles only 1-, 2- and 3-byte sequences;
  every 4-byte sequence - that is, every emoji - falls into `default` and is
  discarded, so the decryption does not round-trip the plaintext. Fix: use
  `util.TextDecoder`.
- **`HW-14-0025`** (B/medium, confirmed): the decrypted JSON is unwrapped with
  fixed `slice(16, len - 4)` offsets on a double-stringified string, so any
  quote, backslash or newline in the message shifts the offsets and the bubble
  renders corrupted; and only `message[0]` is ever encrypted, so multi-span
  input is silently truncated. Fix: join the span values and encrypt the text
  itself, which removes the need to unwrap at all.
- **`HW-14-0001`** (E/low, confirmed; systematic): four social project trees in
  this industry list files their zips do not contain. This sample's tree is
  correct, but do not assume that of its siblings.
- Unfiled, worth knowing: `generateKeyPairSync` on the UI thread per message;
  `handlecCrypto` is fired from `sendMessage` without `await` and has no
  `try/catch`, so a `doFinal` rejection surfaces as an unhandled promise
  rejection rather than a message the user can act on.

## References

- `huawei_industry_tree/14_social_communication/docs/11_encryp_chat.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/encryp_chat-0000002291169549
- `documentation/harmonyos-guides/04_system/crypto-sm2-asym-encrypt-decrypt.md` - the SM2 encrypt/decrypt flow and the `'SM2_256|SM3'` string
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/crypto-sm2-asym-encrypt-decrypt
- `documentation/harmonyos-references/03_system/js-apis-cryptoframework.md` - `createAsyKeyGenerator`, `createCipher`, `CryptoMode`, `DataBlob`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-cryptoframework
- `documentation/harmonyos-references/02_application-framework/js-apis-util.md` - `TextDecoder.create` / `decodeToString` and `Base64Helper`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-util
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-richeditor.md` - `getSpans`, `RichEditorTextSpanResult`, `deleteSpans`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-richeditor
- `documentation/harmonyos-guides/03_application-framework/arkts-common-components-richeditor.md` - span structure and the controller lifecycle
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-common-components-richeditor
- `documentation/harmonyos-references/02_application-framework/ts-container-list.md` - `ListScroller.scrollEdge`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-list
- `SOCIAL-12` - the same `RichEditor` input row, used for link detection instead
