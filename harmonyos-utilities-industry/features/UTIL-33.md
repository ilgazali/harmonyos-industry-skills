---
id: UTIL-33
title: TOTP dynamic password - HMAC-SHA1 over a 30-second time counter, with a Progress bar as the countdown
industry: 15_utilities
doc: huawei_industry_tree/15_utilities/docs/33_dynamic_password_generation.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/dynamic_password_generation-0000002489836613
sample: huawei_industry_tree/15_utilities/downloads/OneTimePassword.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.CoreFileKit", "@kit.CryptoArchitectureKit", "@kit.PerformanceAnalysisKit"]
apis: [cryptoFramework, "cryptoFramework.createMac", "cryptoFramework.createSymKeyGenerator", convertKeySync, initSync, updateSync, doFinalSync, DataView, setBigUint64, Progress, setInterval, "@StorageProp", hilog, window]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-15-0101, HW-15-0108, HW-15-0109, HW-15-0110]
status: verified
---

## When to use

Load this card when the app has to **produce a six-digit code that a server can
verify without any network round trip** - two-step verification, a login
confirmation, an authorisation code for a transfer. The pattern is TOTP
(RFC 6238): both sides hold a shared secret, both derive the same counter from
the wall clock, and the code is a truncation of `HMAC(secret, counter)`.

The whole feature is one utility class and one page. Nothing about it is
HarmonyOS-specific except the crypto calls, so this card is also the reference
for **how to drive `cryptoFramework`'s MAC API synchronously**: create the MAC,
convert raw bytes into a symmetric key, init/update/doFinal. The same three-call
shape covers any HMAC use - request signing, a webhook signature check, an
integrity tag on a local file.

Two things generalise beyond the six digits. First, the **counter is derived,
never stored**: no state survives between two code generations, which is what
lets a fresh install produce a valid code immediately. Second, the countdown is
read back from the clock on every rotation rather than being decremented
forever, so UI drift cannot accumulate.

**This sample's TOTP implementation was checked against RFC 4226/6238 and is
correct** - big-endian 8-byte counter, dynamic truncation on the low nibble of
the last hash byte, 31-bit mask, modulo, left zero-pad. There are no defects to
work around; the caveats in Pitfalls are about the demo scaffolding, not the
algorithm.

## Feature checklist

- The page shows an avatar, a six-digit code in large type, and a thin progress
  bar under it.
- The code is derived from a shared secret and the current 30-second time
  window; it does not come from the network.
- The bar drains over the 30-second window and resets when the window rolls.
- On rollover a new code appears and a toast confirms the refresh.
- The code always has six characters - values below 100000 are left-padded with
  zeros rather than shown short.
- Leaving the page stops the timer; returning re-syncs the code and the
  remaining time against the real clock.
- The second bottom tab is a placeholder: tapping it raises a "demo only" toast
  and the content does not change.

## Architecture

One `entry` module, one page, one utility class. There is no model layer and no
storage: the code is a pure function of `(SECRET, Date.now())`.

```
entry/src/main/ets
├── constants/Constants.ets        SECRET, COUNTDOWN_PERIOD_MS (30000), HEADER_TEXT_PADDING
├── entryability/EntryAbility.ets  full-screen layout, avoid areas -> AppStorage, light colour mode
├── entrybackupability/
├── pages/TokenPage.ets            @Entry: the card, the Progress bar, the 50 ms tick
└── utils
    ├── HMACUtil.ets               the whole TOTP algorithm, 95 lines, all static
    └── LogUtil.ets                hilog wrapper
```

The documented tree matches the zip exactly.

**The design decision worth copying** is that `HMACUtil` is stateless. Every
method is `static`, nothing is cached between calls, and the time counter is
recomputed from `Date.now()` on each generation. That has a practical
consequence: the page can call `generateTOTP` at any moment - on entry, on
resume, on a tick - and always gets the code that is valid *now*, with no
"stale instance" failure mode. The countdown is exposed by the same class
(`getRemainingMilliseconds`) computed from the same clock arithmetic, so the
bar and the code can never disagree about where the window boundary is.

The counterpart worth noticing is what the class does **not** do: it never
stores the secret, never talks to the network, and never persists anything. In
a real product only the secret provisioning changes - the algorithm stays
exactly this.

## Implementation steps

1. **Hold the shared secret as Base32.** The sample uses the RFC test vector
   `'JBSWY3DPEHPK3PXP'` in `Constants.ets`; in production it arrives from the
   server during enrolment and belongs in a key store, not a source constant.
2. **Decode Base32 to bytes yourself.** There is no Base32 helper in the kit -
   `util.Base64Helper` is a different alphabet. Accumulate 5 bits per character
   and emit a byte whenever 8 bits are available.
3. **Convert the bytes into an HMAC symmetric key** with
   `createSymKeyGenerator('HMAC')` and `convertKeySync({ data })`. `'HMAC'`
   here is the *key algorithm*, not the digest - the digest is chosen when the
   MAC is created.
4. **Compute the counter as `floor(epochSeconds / timeStep)`** and write it
   into an 8-byte buffer **big-endian**: `setBigUint64(0, BigInt(n), false)`.
   The `false` is the load-bearing argument; little-endian here yields a code
   that no verifier will accept.
5. **Run the MAC synchronously**: `createMac('SHA1')`, `initSync(symKey)`,
   `updateSync({ data: counter })`, `doFinalSync().data`. SHA1 is not a
   weakness here - HOTP is specified on HMAC-SHA1 and interoperability requires
   it.
6. **Truncate dynamically**: offset = low nibble of the last byte, take four
   bytes from that offset, clear the top bit of the first one, modulo `10^digits`,
   then `padStart(digits, '0')`.
7. **Drive the UI from the clock, not from a counter.** Tick every 50 ms for a
   smooth bar, but when the window rolls, re-read the remaining milliseconds
   from `Date.now()` instead of resetting to 30000.
8. **Clear the interval in `onPageHide`** and re-arm it in `onPageShow`, so a
   backgrounded app is not ticking 20 times a second.

## Verified snippets

All snippets are from `OneTimePassword.zip`. All are **as shipped** - the
implementation is correct as written.

**The algorithm, top to bottom — `entry/src/main/ets/utils/HMACUtil.ets`** (as shipped)

```typescript
import { cryptoFramework } from '@kit.CryptoArchitectureKit';

export class HMACUtil {
  static generateTOTP(secret: string, timeStep: number = 30, digits: number = 6): string {
    // Step 1: Base32 Decoding Key
    let secretData = new Uint8Array(HMACUtil.decodeBase32(secret));
    let symKey = HMACUtil.genSymKeyByData(secretData);

    // Step 2: Calculate time counter
    let timeCounter = HMACUtil.calculateTimeCounter(timeStep);

    // Step 3: Calculated using HMAC-SHA1
    let hmac = cryptoFramework.createMac('SHA1');
    hmac.initSync(symKey);
    hmac.updateSync({ data: timeCounter });
    let hash = hmac.doFinalSync().data;

    // Step 4: Obtain the verification code
    return HMACUtil.genOTPByHash(hash, digits);
  }

  static genSymKeyByData(secret: Uint8Array) {
    let aesGenerator = cryptoFramework.createSymKeyGenerator('HMAC');
    let symKeyBlob: cryptoFramework.DataBlob = { data: secret };
    let symKey = aesGenerator.convertKeySync(symKeyBlob);
    return symKey;
  }

  static calculateTimeCounter(timeStep: number): Uint8Array {
    let timeWindow = Math.floor(Date.now() / 1000 / timeStep);
    // Convert to an 8-byte big-endian data.
    let timeCounter = new DataView(new ArrayBuffer(8));
    timeCounter.setBigUint64(0, BigInt(timeWindow), false);
    return new Uint8Array(timeCounter.buffer);
  }
}
```

**Three calls carry the design.** `createSymKeyGenerator('HMAC')` +
`convertKeySync` is the only route from raw bytes to a `SymKey` the MAC will
accept - you cannot hand `cryptoFramework` a `Uint8Array` directly, and
generating a key instead of converting one would produce a secret the server
does not share. `createMac('SHA1')` picks the digest; the key algorithm string
and the digest string are independent, which is the part most first
implementations get wrong. `setBigUint64(..., false)` is the byte order: RFC
4226 specifies the counter as an 8-byte big-endian integer, and `false` means
"not little-endian".

The `Sync` variants are used throughout. That is a deliberate fit for this
workload - the input is 8 bytes, the cost is microseconds, and doing it
synchronously keeps `generateTOTP` a plain function that a `@State` assignment
can call inline.

**Base32 decode and dynamic truncation — same file** (as shipped)

```typescript
static decodeBase32(originKey: string): Uint8Array {
  let base32Chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ234567';
  let charMap = new Map<string, number>();
  for (let i = 0; i < base32Chars.length; i++) {
    charMap.set(base32Chars.charAt(i), i);
  }
  let bits = 0, value = 0, result: number[] = [];
  originKey = originKey.replace(/[^A-Z2-7]/gi, '').toUpperCase();
  for (let i = 0; i < originKey.length; i++) {
    let c = originKey[i];
    if (!charMap.has(c)) {
      continue;
    }
    value = (value << 5) | charMap.get(c)!;
    bits += 5;
    if (bits >= 8) {
      result.push((value >> (bits - 8)) & 0xFF);
      bits -= 8;
    }
  }
  return new Uint8Array(result);
}

static genOTPByHash(hash: Uint8Array, digits: number): string {
  let offset = hash[hash.length - 1] & 0x0F; // The last one takes the lower 4 bits.
  let binary = ((hash[offset] & 0x7F) << 24) | ((hash[offset + 1] & 0xFF) << 16) |
    ((hash[offset + 2] & 0xFF) << 8) | (hash[offset + 3] & 0xFF);
  // Obtain the verification code by taking the modulus.
  let otp = binary % Math.pow(10, digits);
  return otp.toString().padStart(digits, '0');
}
```

**`& 0x7F` on the first byte is not cosmetic.** It clears the sign bit so the
assembled 32-bit value is always a positive 31-bit integer; without it the
shift produces a negative number in JavaScript and the modulo yields a negative
"code". Likewise `padStart` is what keeps the code six characters long - one
generation in ten produces a value below 100000, and a verifier comparing
strings would reject `12345` where it expected `012345`.

The decoder's `replace(/[^A-Z2-7]/gi, '')` strips the spaces and `=` padding
that provisioning QR codes and manual entry usually carry, so the same function
accepts `JBSW Y3DP EHPK 3PXP` and the compact form. Because the alphabet is
exactly the RFC 4648 Base32 alphabet, keys copied out of any standard
authenticator enrolment work unchanged.

**The countdown loop — `entry/src/main/ets/pages/TokenPage.ets`** (as shipped)

```typescript
@State token: string = '';
@State countDown: number = COUNTDOWN_PERIOD_MS;   // 30000
private intervalId?: number;

refreshToken(): void {
  let lastToken = this.token;
  this.token = HMACUtil.generateTOTP(SECRET);
  this.countDown = HMACUtil.getRemainingMilliseconds();   // re-read from the clock
  if (this.token !== lastToken) {
    this.getUIContext().getPromptAction().showToast({
      message: $r('app.string.toast_refresh')
    });
  }
}

onPageShow(): void {
  this.refreshToken();
  this.intervalId = setInterval(() => {
    this.countDown -= 50;
    if (this.countDown <= 0) {
      this.refreshToken();
    }
  }, 50);
}

onPageHide(): void {
  if (this.intervalId !== undefined) {
    clearInterval(this.intervalId);
    this.intervalId = undefined;
  }
}
```

**Two clocks, one authority.** The 50 ms tick exists only so the `Progress` bar
moves smoothly; it is allowed to drift, because at every window boundary
`refreshToken` throws its accumulated error away and takes the remaining time
from `getRemainingMilliseconds()`, which is computed from `Date.now()` against
the same window arithmetic the code itself uses. That is why the bar cannot
slowly desynchronise from the code even after hours on screen.

The `lastToken` comparison is what stops the toast firing on every entry to the
page: `onPageShow` always calls `refreshToken`, but inside the same 30-second
window the code is unchanged and the toast is suppressed.

The bar itself is a plain `Progress` fed the two numbers directly:

```typescript
Progress({
  value: this.countDown,
  total: COUNTDOWN_PERIOD_MS
})
  .height(4)
  .color($r('app.color.main_theme'));
```

No animation attribute is needed - re-rendering 20 times a second is the
animation.

## Permissions & config

**None.** The sample declares no `requestPermissions`. TOTP is offline
arithmetic: no network, no storage, no device identifiers.

`deviceTypes` is `phone`, `tablet`, `2in1`. `EntryAbility` pins light mode with
`setColorMode(COLOR_MODE_LIGHT)` and publishes the top/bottom avoid-area heights
into `AppStorage`, which the page consumes as `@StorageProp` and converts with
`px2vp` at the point of use - the same shape as `TOUR-03`, and the safer one,
because a missing key falls back to a typed `0`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **Device and server clocks must agree.** TOTP has no negotiation: a device
  more than one window out of step produces codes the server rejects.
  Production verifiers usually accept ±1 window; the client side cannot fix
  skew on its own, so surface a "check your device time" hint on repeated
  rejection.
- The secret is a source constant. Anyone with the APP has the seed, so this
  build authenticates nothing - it demonstrates the algorithm. Real enrolment
  delivers the secret over an authenticated channel and stores it in a
  hardware-backed key store.
- `digits` and `timeStep` are parameters with RFC defaults (6 / 30 s) but the
  page never passes them; changing them requires the verifier to change too.
- The second bottom tab holds an empty `TabContent`. `onContentWillChange`
  returns `false` on purpose and shows a "demo only" toast, so the tab bar
  highlight moves but the content does not - intentional here, unlike the same
  construct in `TOUR-03` where it silently killed three real tabs.

## Pitfalls

- **No defects were found in this sample.** The TOTP path was read line by line
  against RFC 4226 (HOTP truncation) and RFC 6238 (time counter): Base32
  decoding, big-endian counter, HMAC-SHA1, dynamic offset, 31-bit mask, modulo
  and zero-padding are all correct, and the countdown re-syncs from the wall
  clock on every rotation. No finding is filed against `UTIL-33`.
- Watch the two *scaffolding* limits when copying: the hardcoded `SECRET`, and
  the fact that the interval is bound to page visibility only. Both are right
  for a demo and wrong for a product - the secret needs a key store, and a
  long-lived page should also stop ticking when the ability goes to background.
- If you change the digest, change it in one place only. `createSymKeyGenerator('HMAC')`
  takes the *key* algorithm and `createMac('SHA1')` the digest; editing the
  first string in the belief that it selects SHA-256 produces a key the MAC
  still accepts and a code that silently differs from the server's.

## References

- `documentation/harmonyos-references/03_system/js-apis-cryptoframework.md` - `createMac`, `initSync`, `updateSync`, `doFinalSync`, `createSymKeyGenerator`, `convertKeySync`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-cryptoframework
- `documentation/harmonyos-guides/04_system/crypto-compute-mac-overview.md` - the MAC computation flow and supported digests
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/crypto-compute-mac-overview
- `documentation/harmonyos-guides/04_system/crypto-sym-key-generation-conversion-spec.md` - the `HMAC` key specification and length rules
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/crypto-sym-key-generation-conversion-spec
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-progress.md` - `Progress` value/total
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-progress
- `huawei_industry_tree/15_utilities/docs/33_dynamic_password_generation.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/dynamic_password_generation-0000002489836613
