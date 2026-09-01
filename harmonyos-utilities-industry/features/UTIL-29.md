---
id: UTIL-29
title: Three RSA modes - PKCS1 for short strings, segmented PKCS1 for long ones, PKCS1_OAEP for the rest
industry: 15_utilities
doc: huawei_industry_tree/15_utilities/docs/29_rsa_encryption_and_decryption.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/rsa_encryption_and_decryption-0000002465072997
sample: huawei_industry_tree/15_utilities/downloads/cipherRSA.zip
kits: ["@kit.CryptoArchitectureKit", "@kit.ArkTS", "@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit"]
apis: ["cryptoFramework.createAsyKeyGenerator", "cryptoFramework.createAsyKeyGeneratorBySpec", "cryptoFramework.createCipher", convertKey, convertKeySync, generateKeyPairSync, "Cipher.init", "Cipher.initSync", "Cipher.doFinal", "Cipher.doFinalSync", setCipherSpec, getCipherSpec, CipherSpecItem, RSAKeyPairSpec, "util.Base64Helper", "buffer.from", Navigation, NavPathStack, NavDestination, window]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-15-0065, HW-15-0066, HW-15-0067]
status: verified-with-fixes
---

## When to use

**Load this card when you have to move a string through RSA** and need to pick
one of the three shapes the crypto framework offers: plain `PKCS1` with a
fixed key pair, `PKCS1` applied block by block so the payload can exceed the
key size, and `PKCS1_OAEP` with a caller-supplied encoding parameter. The
sample is a three-way menu over exactly those three, with the same
encrypt/decrypt screen behind each entry.

The transferable content is not "how to call `doFinal`" - it is **the size
budget**. RSA cannot encrypt more than the key allows: a 512-bit key under
PKCS1 takes 53 bytes, a 2048-bit key under OAEP/SHA256 takes 190. Every real
RSA feature therefore has to choose between a bigger key, segmentation, or the
usual production answer - encrypt a symmetric key with RSA and the payload
with AES. This sample shows the first two and leaves the third to the reader.

The second transferable piece is the **key provenance**. Mode 1 carries a
Base64 key pair as string constants, mode 2 generates a fresh pair at module
load, mode 3 rebuilds a fixed pair from `n`/`e`/`d` big integers on every call.
Those three choices have very different lifetimes, and the sample presents them
behind one identical UI without saying so - which is what `HW-15-0067` is
about. Read it before you copy mode 2.

## Feature checklist

- A home page listing three RSA modes: PKCS1 短数据 (short data), PKCS1
  长数据 (long data), and PKCS1_OAEP.
- Tapping a mode opens a page with an 加密 (encrypt) and a 解密 (decrypt)
  button.
- The encrypt screen takes plaintext in a `TextArea` and shows Base64
  ciphertext, selectable and copyable in-app.
- The decrypt screen takes Base64 ciphertext and shows the recovered string.
- Empty input raises a "This message is null." toast instead of running the
  cipher.
- Unparseable ciphertext produces the fixed message 解密内容输入错误，无法解析
  (the decryption input is wrong and cannot be parsed) rather than a crash.
- A reset button clears the result panel on both screens.
- Round trip is exact for the plaintext each mode is sized for, including
  non-ASCII - which is where `HW-15-0065` bites.

## Architecture

One `entry` module. The crypto lives in one file, the UI in four, and routing
goes through `Navigation` + a router map rather than `router.pushUrl`.

```
entry/src/main/ets
├── common
│   ├── Encrypt.ets       encrypt screen: TextArea + result panel + the mode switch
│   └── Decrypt.ets       decrypt screen: the same layout, err-checked callbacks
├── entryability/EntryAbility.ets   full-screen window, avoid areas -> AppStorage
├── model
│   ├── CipherModel.ts    all three modes: CipherModel1/2/3 + the two converters
│   └── Logger.ts         hilog wrapper
└── pages
    ├── Index.ets         @Entry, Navigation host, the three mode cards, PageData
    ├── RSA.ets           NavDestination: encrypt/decrypt choice for one mode
    └── RSAPage.ets       NavDestination: hosts Encrypt or Decrypt by a flag
```

The documented tree matches the zip apart from one letter: the document writes
`entryAbility`, the zip ships `entryability`. Cosmetic, but it is the directory
name `module.json5` points `srcEntry` at, so do not "fix" it in either
direction without changing both.

**The design decision worth copying** is that all three modes expose the same
two-method surface, `rsaEncrypt(message, callback)` and
`rsaDecrypt(message, callback)`, with the identical `EncryptCallback` type:

```typescript
type EncryptCallback = (error: Error | null, result?: string) => void;
```

That is what lets one `Encrypt` component serve three different algorithms with
a string discriminator (`this.rsa === 'RSA1' | 'RSA2' | 'RSA3'`) and nothing
else. Strings in, strings out, Base64 on the wire in every mode - so the screens
never learn what a `DataBlob` is. If you add a fourth mode, you add a class and
one `else if`.

**The decision worth avoiding** sits three lines above it:

```typescript
const asyKeyGenerator = cryptoFramework.createAsyKeyGenerator("RSA1024");
export const keyPair = asyKeyGenerator.generateKeyPairSync();
```

A module-level key pair is process-scoped state dressed up as a constant. Mode
2's ciphertext survives exactly as long as the app process, while mode 1's
survives forever, and the UI presents both as paste-in/paste-out tools
(`HW-15-0067`).

## Implementation steps

1. **Pick the mode from the payload size, not from taste.** RSA512|PKCS1 takes
   53 plaintext bytes, RSA1024|PKCS1 takes 117, RSA2048|PKCS1_OAEP|SHA256 takes
   190. Beyond that you segment or you switch to a hybrid scheme.
2. **Decode the Base64 key with `util.Base64Helper`, then hand the bytes to
   `convertKey`** as a `DataBlob`. `convertKeySync(pubKeyBlob, null)` for a
   public key, `convertKey(null, priKeyBlob, cb)` for a private one - the two
   parameters are positional and passing `null` for the other one is how you
   say "I only have this half".
3. **Init with the matching `CryptoMode`** and the half of the pair that mode
   needs: `ENCRYPT_MODE` with `keyPair.pubKey`, `DECRYPT_MODE` with
   `keyPair.priKey`.
4. **Check `err` before touching `data` in every `doFinal` callback**
   (`HW-15-0066`). The OAEP path in `CipherModel3` does not, and an oversize
   input turns a clean crypto error into a `TypeError` on `data.data`.
5. **Check `err` in the UI callbacks too** (`HW-15-0066`). `Decrypt.ets` does;
   `Encrypt.ets` does not, and renders the literal string `undefined` as the
   ciphertext when the model reports failure.
6. **Segment on 64-byte plaintext blocks and 128-byte ciphertext blocks for a
   1024-bit key.** The ciphertext block size is fixed by the key
   (bits / 8); the plaintext block size only has to stay under the padding
   limit.
7. **Decode the recovered bytes with `buffer.from(bytes).toString('utf-8')`**,
   never with a hand-written switch on the lead byte (`HW-15-0065`).
8. **Decide the key lifetime explicitly** (`HW-15-0067`). A generated pair is
   session-scoped; if the ciphertext outlives the session, the key has to be
   persisted or derived.
9. **Set the OAEP `pSource` on both sides.** Encrypt and decrypt must agree on
   the encoding parameter or the unpad step fails; the sample sets the same
   `[1, 2, 3, 4]` in both.

## Verified snippets

All snippets are from `cipherRSA.zip`. Corrected forms are marked.

**Mode 1, fixed key pair, callback chain — `entry/src/main/ets/model/CipherModel.ts`** (as shipped)

```typescript
const RSA512_PRIMES_2: string = 'RSA512|PRIMES_2';
const RSA512_PKCS1: string = 'RSA512|PKCS1';

rsaEncrypt(message: string, callback: EncryptCallback): void {
  // 指定“RSA密钥类型”RSA512和“素数个数”PRIMES_2来创建RSA-512密钥生成器
  let rsaGenerator = cryptoFramework.createAsyKeyGenerator(RSA512_PRIMES_2);
  // 传入“非对称密钥类型”RSA512和“填充模式 PKCS1”创建RSA-PKCS1加密器
  let cipher = cryptoFramework.createCipher(RSA512_PKCS1);
  let that = new util.Base64Helper();
  let pubKey = that.decodeSync(RSA_ENCRYPT_KEY);
  if (!pubKey) {
    throw new Error("Failed to decode Base64 public key");
  }
  let pubKeyBlob: cryptoFramework.DataBlob = { data: pubKey };
  let keyPair = rsaGenerator.convertKeySync(pubKeyBlob, null)
  if (!keyPair) {
    throw new Error("Failed to Get keyPair");
  }
  cipher.init(cryptoFramework.CryptoMode.ENCRYPT_MODE, keyPair.pubKey, null, (err, data) => {
    if (err) {
      Logger.error(TAG, 'init encrypt: error.' + (err as BusinessError).code);
      callback(new Error('Failed to initialize encrypt'));
      return;
    }
    let input: cryptoFramework.DataBlob = { data: stringToUint8Array(message) };
    cipher.doFinal(input, (err, data) => {
      if (err) {
        Logger.error(TAG, 'doFinal encrypt: error.' + (err as BusinessError).code);
        callback(new Error('Failed to encrypt message'));
        return;
      }
      let result = that.encodeToStringSync(data.data);
      callback(null, result);
    })
  })
}
```

**Two strings carry the whole configuration.** `'RSA512|PRIMES_2'` describes
the *key* - bit length and prime count - and goes to the generator;
`'RSA512|PKCS1'` describes the *transformation* - key type plus padding - and
goes to the cipher. They are separate objects and the specifier grammar for
each is different, which is the single most common source of "invalid
parameter" errors when adapting this code to another key size.

`convertKeySync(pubKeyBlob, null)` returns a `KeyPair` whose `priKey` is
absent. That is fine for encryption and is why the encrypt path can ship a
public key only. Note the shape of the callback chain: `init` is asynchronous
and `doFinal` is nested inside its callback, because a cipher object must not
be used before its init resolves.

**The UTF-8 decoder — same file** (corrected, see `HW-15-0065`)

```typescript
// Uint8Array转字符串方法
function uint8ArrayToString(array: Uint8Array): string {
  try {
    // FIX: the shipped body is a hand-rolled switch over `character >> 4` with
    // cases 0-7 (ASCII), 12-13 (2-byte) and 14 (3-byte) only. Lead bytes
    // 0xF0-0xF4 - every 4-byte sequence, i.e. all emoji and the CJK
    // supplementary planes - fall into `default: break;` and are dropped one
    // byte at a time, so the characters vanish from the decrypted string
    // without any error.
    return buffer.from(array).toString('utf-8');
  } catch (err) {
    Logger.error(TAG, 'Error converting Uint8Array to string: ' + (err as Error).message);
    throw new Error('Failed to convert Uint8Array to string');
  }
}
```

**The sample already contains the correct form.** `CipherModel2.rsaDecrypt`
ends with `let result = buffer.from(decryptText.data).toString('utf-8');` -
one line, full UTF-8, surrogate pairs included. Modes 1 and 3 call the
hand-written decoder instead, so the identical round trip either preserves an
emoji or silently eats it depending on which menu entry you tapped.

The encode direction is fine in all three modes
(`new Uint8Array(buffer.from(str, 'utf-8').buffer)`), which is what makes the
defect so quiet: the ciphertext is correct and complete, the loss happens on
the way back out.

**Mode 2, segmentation — same file** (as shipped)

```typescript
rsaEncryptBySegment(pubKey: cryptoFramework.PubKey,
                    plainText: cryptoFramework.DataBlob): cryptoFramework.DataBlob {
  let cipher = cryptoFramework.createCipher('RSA1024|PKCS1');
  cipher.initSync(cryptoFramework.CryptoMode.ENCRYPT_MODE, pubKey, null);
  let plainTextSplitLen = 64;
  let cipherText = new Uint8Array();
  for (let i = 0; i < plainText.data.length; i += plainTextSplitLen) {
    let updateMessage = plainText.data.subarray(i, i + plainTextSplitLen);
    let updateMessageBlob: cryptoFramework.DataBlob = { data: updateMessage };
    // 将原文按64字符进行拆分，循环调用doFinal进行加密，使用1024bit密钥时，每次加密生成128字节长度的密文。
    let updateOutput = cipher.doFinalSync(updateMessageBlob);
    let mergeText = new Uint8Array(cipherText.length + updateOutput.data.length);
    mergeText.set(cipherText);
    mergeText.set(updateOutput.data, cipherText.length);
    cipherText = mergeText;
  }
  return { data: cipherText };
}
```

**The two block sizes are not the same number and must not be derived from
each other.** 64 is a plaintext chunk chosen to sit under the PKCS1 limit for a
1024-bit key (117 bytes); 128 is the ciphertext chunk and is fixed by the key -
1024 / 8 - so the decrypt loop uses `cipherTextSplitLen = 128` and has no
freedom. Change the key size and both constants move, in different ways.

Note that this is `doFinal` per block, not `update` + one `doFinal`: RSA has no
streaming mode, so each chunk is an independent encryption and the output is a
plain concatenation. The `mergeText` dance is just "append to a Uint8Array";
the reason it is written out longhand is that typed arrays have no `concat`.

`initSync` is called once outside the loop - the cipher object is reusable
across `doFinalSync` calls, and re-initialising per block would be pure waste.

**Mode 3, OAEP — same file** (corrected, see `HW-15-0066`)

```typescript
let cipher = cryptoFramework.createCipher("RSA2048|PKCS1_OAEP|SHA256|MGF1_SHA1");
// RSA加解密PKCS1-OAEP模式填充字节流P。
let pSource = new Uint8Array([1, 2, 3, 4]);
let input: cryptoFramework.DataBlob = { data: new Uint8Array(buffer.from(plan, 'utf-8').buffer) };
let keyPair = rsaGeneratorSpec.generateKeyPairSync();
cipher.initSync(cryptoFramework.CryptoMode.ENCRYPT_MODE, keyPair.pubKey, null);
// get和set操作可以放在Cipher对象init之后，此处对cipher进行set和get操作。
cipher.setCipherSpec(cryptoFramework.CipherSpecItem.OAEP_MGF1_PSRC_UINT8ARR, pSource);
let retP = cipher.getCipherSpec(cryptoFramework.CipherSpecItem.OAEP_MGF1_PSRC_UINT8ARR);
if (retP.toString() !== pSource.toString()) {
  Logger.error("ccc error init pSource" + retP);
}
cipher.doFinal(input, (err, data) => {
  if (err) {                                     // FIX: absent in the sample
    Logger.error(TAG, 'doFinal encrypt: error.' + (err as BusinessError).code);
    callback(new Error('Failed to encrypt message'));
    return;
  }
  let result = that.encodeToStringSync(data.data);
  callback(null, result);
})
```

**Four fields in one specifier string.** `RSA2048` is the key,
`PKCS1_OAEP` the padding, `SHA256` the OAEP digest and `MGF1_SHA1` the mask
generation function with *its own, different* digest. The two hashes are
independent by design and both sides must agree on both; the usable plaintext
is `256 - 2 * 32 - 2 = 190` bytes, derived from the key size and the SHA256
digest length.

`pSource` is the optional OAEP label. It is set through `setCipherSpec` after
`initSync` (the comment notes it also works before) and read back through
`getCipherSpec` purely as a self-check. Encrypt and decrypt must set the same
label or the unpad step rejects the message with a generic error.

Without the `err` guard, typing 191 characters makes `data` undefined, the
`data.data` dereference throws inside the framework's callback, and the
user-facing callback never fires at all - the screen simply keeps showing the
previous result. That is the difference between "your input is too long" and
"the button is broken".

## Permissions & config

**None.** `module.json5` declares no `requestPermissions` - RSA runs entirely
in-process and the sample never touches the network or storage.

`deviceTypes` is `["phone"]` only, the narrowest of the four utilities samples
reviewed here. Routing is declared in `$profile:router_map` with two entries,
`RSA` and `RSAPage`, each naming a `@Builder` function
(`cipherRSABuilder`, `cipherRSAPageBuilder`); the pages themselves are
`NavDestination`s pushed with `pageStack.pushPathByName(name, new PageData(...))`.
`PageData` carries the two-field payload - the mode string and an
encrypt/decrypt boolean - and each destination reads it back in `onReady` from
`ctx.pathInfo.param`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **The keys are demo keys.** `RSA_ENCRYPT_KEY` and `RSA_DECRYPT_KEY` are
  Base64 constants in the source. A shipped app would put the private half in
  HUKS, not in a string literal.
- Mode 1's 512-bit key is below every current recommendation. It is here
  because it makes the 53-byte size limit easy to hit and demonstrate; do not
  carry the number into production.
- Mode 3 rebuilds the same 2048-bit pair from hard-coded `n`/`e`/`d` big
  integers on *every* encrypt and every decrypt call. It is deterministic - so
  the round trip survives restarts - but it is also two full key
  reconstructions per operation.
- `CipherModel3.rsaDecrypt` re-encodes the incoming Base64 *string* into
  `input` and compares the decrypted bytes against it. The comparison is
  meaningless (it can only match if the plaintext happened to equal its own
  Base64) and only feeds a log line, but it is dead weight in a path that is
  otherwise correct.
- The result panel is capped at `maxLines(12)` with an ellipsis, so long
  ciphertext is visually truncated. `copyOption(CopyOptions.InApp)` still
  copies the whole string.

## Pitfalls

- **`HW-15-0065`** (B/medium, confirmed): `uint8ArrayToString` handles only
  1-, 2- and 3-byte UTF-8 sequences; lead bytes `0xF0`-`0xF4` hit the `default`
  branch and every byte of a 4-byte sequence is skipped, so emoji and
  supplementary-plane characters disappear from modes 1 and 3 decryptions with
  no error. Fix: use `buffer.from(bytes).toString('utf-8')`, exactly as
  `CipherModel2` already does.
- **`HW-15-0066`** (B/medium, confirmed): the `err` argument is ignored twice -
  `CipherModel3`'s `doFinal` callback dereferences `data.data` unconditionally,
  so oversize OAEP input throws and the user callback never runs; and the RSA1
  and RSA2 buttons in `Encrypt.ets` assign `${result}` without checking `err`,
  rendering the literal `undefined`. Fix: guard `err` before `data`, in the
  model and in the UI. `Decrypt.ets` shows the right shape.
- **`HW-15-0067`** (B/low, confirmed): mode 2's key pair comes from a
  module-level `generateKeyPairSync()`, so it is new on every launch and
  yesterday's ciphertext can never be decrypted - while mode 1, behind an
  identical UI, uses fixed keys and works forever. Fix: persist the pair, or
  label the mode as session-scoped in the UI.

## References

- `documentation/harmonyos-references/03_system/js-apis-cryptoframework.md` -
  `createAsyKeyGenerator`, `createCipher`, `convertKey`, `doFinal`,
  `setCipherSpec`, `CipherSpecItem`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-cryptoframework
- `documentation/harmonyos-guides/04_system/crypto-asym-encrypt-decrypt-spec.md` -
  the padding-mode matrix and the per-key plaintext size limits
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/crypto-asym-encrypt-decrypt-spec
- `documentation/harmonyos-guides/04_system/crypto-rsa-asym-encrypt-decrypt-pkcs1.md` -
  mode 1's flow end to end
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/crypto-rsa-asym-encrypt-decrypt-pkcs1
- `documentation/harmonyos-guides/04_system/crypto-rsa-asym-encrypt-decrypt-by-segment.md` -
  mode 2's segmentation, including the block-size arithmetic
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/crypto-rsa-asym-encrypt-decrypt-by-segment
- `documentation/harmonyos-guides/04_system/crypto-rsa-asym-encrypt-decrypt-pkcs1_oaep.md` -
  mode 3, `pSource` and the two OAEP digests
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/crypto-rsa-asym-encrypt-decrypt-pkcs1_oaep
- `documentation/harmonyos-guides/04_system/crypto-generate-asym-key-pair-from-key-spec.md` -
  `createAsyKeyGeneratorBySpec` and `RSAKeyPairSpec`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/crypto-generate-asym-key-pair-from-key-spec
- `documentation/harmonyos-guides/04_system/crypto-asym-key-generation-conversion-spec.md` -
  the `RSA512|PRIMES_2` generator specifier grammar
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/crypto-asym-key-generation-conversion-spec
- `documentation/harmonyos-references/02_application-framework/js-apis-buffer.md` -
  `buffer.from` and `toString('utf-8')`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-buffer
- `UTIL-30` - the sibling crypto card, HMAC over the same `cryptoFramework`
