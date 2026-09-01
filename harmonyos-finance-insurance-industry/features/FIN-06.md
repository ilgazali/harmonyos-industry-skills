---
id: FIN-06
title: App lock - gesture and text password, encrypted and persisted
industry: 07_finance_insurance
doc: huawei_industry_tree/07_finance_insurance/docs/06_custom_app_lock.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/custom_app_lock-0000002319233110
sample: huawei_industry_tree/07_finance_insurance/downloads/CustomAppLock.zip
kits: ["@kit.ArkUI", "@kit.CryptoArchitectureKit", "@kit.ArkData", "@kit.ArkTS"]
apis: [PatternLock, PatternLockController, onPatternComplete, cryptoFramework.createAsyKeyGenerator, convertKey, createCipher, Cipher.init, Cipher.doFinal, preferences.getPreferences, Preferences.put, Preferences.get, Preferences.flush]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-07-0023, HW-07-0024, HW-07-0025, HW-07-0026, HW-07-0027, HW-07-0053]
status: verified-with-fixes
---

## When to use

Load this card when an app needs its **own lock screen** - a gesture pattern or
a PIN the user must clear before seeing balances, holdings or personal financial
detail. The `PatternLock` component and the setting/confirm/login page flow are
worth taking.

**The storage half of this practice should not be copied.** Five findings, one
of them a blocker, and all five are about how the credential is protected. Read
the pitfalls before the snippets: the UI is a good starting point, the security
design is not.

## Feature checklist

- A gesture lock the user draws, confirmed by drawing it twice.
- A text password as an alternative.
- Either can be enabled independently; both are remembered across launches.
- On launch, the lock screen appears before the account screen.
- The password entry screen is excluded from screen recording.

## Architecture

Single `entry` module, one page per step of the flow.

```
entry/src/main/ets
├── pages/MinePage.ets              home; decides whether a lock is set
├── pages/PromptPage.ets            gesture-lock intro
├── pages/ImageSettingPage.ets      draw the gesture to set it
├── pages/ImageLoginPromptPage.ets  gesture-lock login intro
├── pages/ImageLoginPage.ets        draw the gesture to unlock
├── pages/TextSettingPage.ets       set the text password
├── pages/TextLoginPage.ets         enter the text password
├── pages/PassSettingPage.ets       change an existing password
├── utils/EncryptionUtil.ets        RSA encrypt/decrypt helpers
└── utils/PreferencesUtils.ets      key-value persistence
```

Two flags in preferences (`isSetText`, `isSetImage`) record which locks are
configured; two more keys (`TextPass`, `ImagePass`) hold the credentials.

## Implementation steps

1. **Draw the lock with `PatternLock`** and collect the result in
   `onPatternComplete`.
2. **Convert the pattern to a comparable value** - the callback gives an
   `Array<number>` of node indices.
3. **Store a salted hash of it, not the value itself** (`HW-07-0024`).
4. **Put the credential in the Asset Store Kit**, not Preferences
   (`HW-07-0025`).
5. **Never hardcode key material** (`HW-07-0023`).
6. **Treat "no credential stored" as a real state** and handle it before
   attempting any decryption (`HW-07-0026`).

## Verified snippets

Snippets are from `CustomAppLock.zip`. The UI ones are as shipped; the storage
ones are replaced, because the shipped design should not be reproduced.

**The gesture lock — `pages/ImageSettingPage.ets`** (as shipped)

```typescript
PatternLock(this.patternLockController)
  .onPatternComplete((input: Array<number>) => {
    // input holds the node indices the user connected, 0..8 for a 3x3 grid,
    // in the order they were touched
  })
```

`PatternLock` gives you the pattern as an ordered array of node indices, so the
value to verify is just `input.join('')` or similar. `PatternLockController`
lets the page reset the drawing between the two confirmation passes.

The document notes that **the password entry screen cannot be screen-recorded** -
that protection comes from the component, not from code you write.

**Storing the credential** (replaced - see `HW-07-0023`, `HW-07-0024`, `HW-07-0025`)

The shipped sample encrypts the password with an RSA key pair whose **private
key is a byte-array literal in `EncryptionUtil.ets`**, writes the ciphertext to
`Preferences`, and decrypts it back to plaintext to compare on login. Do none of
those things. The correct shape:

```typescript
// verification, not recovery: store a salted digest
import { cryptoFramework } from '@kit.CryptoArchitectureKit';

async function hashPattern(pattern: string, salt: Uint8Array): Promise<string> {
  const md = cryptoFramework.createMd('SHA256');
  await md.update({ data: new Uint8Array(buffer.from(pattern, 'utf-8').buffer) });
  await md.update({ data: salt });
  const digest = await md.digest();
  return uint8ArrayToHexString(digest.data);
}

// on set:    generate a random salt, store salt + digest
// on unlock: hash the attempt with the stored salt, compare digests
// the original pattern is never recoverable and never needs to be
```

Put the salt and digest in the **Asset Store Kit**, which is the platform
component for credentials - it binds the item to the device and can gate access
on user authentication. `Preferences` is an ordinary file in the sandbox and
offers persistence, not protection.

If a credential genuinely must be reversible - which an app lock's does not -
the key belongs in the Universal Keystore, generated on device, never in source.

**Reading a credential that may not exist** (corrected, see `HW-07-0026`)

```typescript
// FIX: the shipped getString returns the literal string 'null' when a key is
// absent, and nothing checks for it - so MinePage decrypts 'null' on first run,
// and 'null' being truthy makes isSetText true when nothing was ever set.
async aboutToAppear() {
  const storedText = await Store.get('TextPass');      // undefined when absent
  this.isSetText = storedText !== undefined;
  this.isSetImage = (await Store.get('ImagePass')) !== undefined;
  // no decryption here at all - the home page has no business holding credentials
}
```

Two rules this makes concrete: **an absent value must be distinguishable from a
present one** - a truthy sentinel string is the bug - and **a page that only
needs to know whether a lock exists should not load the credential**.

**Preferences, for things that are not secrets — `utils/PreferencesUtils.ets`** (as shipped)

```typescript
const PREFERENCES_NAME: string = 'myPreferences';

class PreferencesUtils {
  private mPreferences: preferences.Preferences | undefined = undefined;

  async setString(keyString: string, value: string, context: Context) {
    if (this.mPreferences === undefined) {
      this.mPreferences = await preferences.getPreferences(context, PREFERENCES_NAME);
    }
    await this.mPreferences.put(keyString, value);
    await this.mPreferences.flush();     // put() alone does not persist
  }
}
```

The lazy-init-then-reuse shape and the `put` + `flush` pairing are both correct
and worth copying - `put` writes to memory, `flush` commits to disk. Use this
class for the `isSetText` / `isSetImage` flags. Do not use it for the
credentials.

## Permissions & config

**None.** The sample declares no `requestPermissions`. Note that Asset Store
Kit, the store this practice should be using, also needs no permission.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- `PatternLock` is a 3x3 grid; the pattern is an ordered index array.
- The password entry screen cannot be recorded - the document states this.
- `Preferences` values are capped: keys 80 bytes, values 8192 bytes.
- There is no biometric path in this sample. For a finance app, User
  Authentication Kit is the expected companion to a local lock.

## Pitfalls

- **`HW-07-0023` (blocker) — the RSA private key is a literal in the source.**
  Because this is a published sample, that key is public: every app built on it
  that keeps the constants shares one publicly-known key, and the ciphertext it
  protects sits in a readable sandbox file.
- **`HW-07-0024` — the password is stored reversibly and decrypted to compare.**
  `MinePage` decrypts both credentials in `aboutToAppear` merely to hold them, so
  opening the home screen materialises both cleartext passwords with no login
  involved.
- **`HW-07-0025` — credentials live in `Preferences`,** which offers persistence
  and not protection, while the platform ships the Asset Store Kit for exactly
  this.
- **`HW-07-0026` — first run decrypts the string `'null'`** with no guard and no
  `try/catch`, and because `'null'` is truthy the app also concludes a password
  is set when none is.
- **`HW-07-0027` — the document's snippet encrypts one value and stores
  another.** It computes `text`, the ciphertext, then writes `passWord`. Followed
  literally, it persists the plaintext.

## References

- `documentation/harmonyos-references/02_application-framework/ts-basic-components-patternlock.md` - `PatternLock`, `onPatternComplete`, `PatternLockController`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-patternlock
- `documentation/harmonyos-guides/04_system/asset-store-kit-overview.md` - the component credentials belong in
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/asset-store-kit-overview
- `documentation/harmonyos-guides/04_system/asset-store-kit.md` - storing and retrieving a secret
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/asset-store-kit
- `documentation/harmonyos-references/03_system/asset-store-api.md` - the Asset Store API
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/asset-store-api
- `documentation/harmonyos-references/03_system/js-apis-cryptoframework.md` - digests, ciphers, key generation
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-cryptoframework
- `documentation/harmonyos-references/02_application-framework/js-apis-data-preferences.md` - `put`, `get`, `flush`, and the size limits
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-data-preferences
- `FIN-01` - the industry architecture; its login screen is the other place credentials are handled
