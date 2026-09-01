---
id: COMMON-06
title: Huawei Pay checkout - server pre-order, client requestPayment, server-verified result
industry: 19_common_technical_solutions
doc: huawei_industry_tree/19_common_technical_solutions/docs/06_practice-common-huawei-pay-v1.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-common-huawei-pay-v1-0000002004896593
sample: none (architecture practice document, no sample project)
kits: ["@kit.PaymentKit", "@kit.AbilityKit", "@kit.BasicServicesKit"]
apis: ["paymentService.requestPayment", "paymentService.PayResult", "common.UIAbilityContext", "UIContext.getHostContext"]
permissions: []
min_api: n/a
modules: [entry]
findings: [HW-19-0002, HW-19-0003]
status: verified-with-fixes
---

## When to use

Load this card when a merchant application must let the user **pay for a
real-world product or service inside the app** - hotel booking, travel, top-up
and bill payment, retail, catering, transport - and show the result. It covers
the Huawei Pay checkout (收银台) integration: which side creates the order, which
API raises the checkout screen, and which side is allowed to decide that the
payment succeeded.

Do **not** load this card for in-app virtual goods (game currency, props,
memberships, subscriptions). Those are IAP Kit products, and Payment Kit
merchants are registered under the explicitly non-virtual PetalPayMerchant
category - see HW-19-0002.

## Feature checklist

The application and its server must together:

- Create the product order on the **merchant server**, never on the client.
- Call the Huawei Pay pre-ordering API from the server to obtain a `prepayId`.
- Assemble the `orderStr` on the server and return it to the client.
- Raise the Huawei Pay checkout screen from the client with
  `paymentService.requestPayment`.
- Handle the promise rejection path and surface `error.code` to logs.
- Receive the payment result on the **merchant server** through the Huawei Pay
  callback.
- Verify that callback with SM2 signature verification before acting on it.
- Treat the server-side result - not the client promise - as the authoritative
  payment state (HW-19-0003).
- Pick the merchant model up front: direct merchant (商户), platform merchant
  (平台类商户), or service provider (服务商); it decides which pre-ordering API and
  which callback notification apply.

## Architecture

**Merchant models.** Huawei Pay offers three access identities, and the choice
constrains which payment capabilities are available:

| Capability | Direct merchant | Platform merchant | Service provider |
| --- | --- | --- | --- |
| 单次支付 single payment | yes | yes | yes |
| 合单支付 combined payment (one or more orders from different merchants merged into a single payment) | - | yes | - |
| 支付并签约 pay and sign (agreement signed after payment, enabling later automatic deduction) | yes | - | yes |
| 签约代扣 sign and withhold (merchant initiates the agreement, e.g. prepaid utilities, auto top-up) | yes | - | yes |

Rule-based settlement splitting is supported, with configurable recipients and
split ratios.

**Checkout flow** (the document's seven steps, with the responsible side):

1. Client -> merchant server: request creation of the product order.
2. Merchant server -> Huawei Pay: call the pre-ordering API for the chosen
   merchant model (direct merchant, or platform merchant / service provider) to
   get `prepayId`; assemble `orderStr` and return it to the client.
3. Client -> Payment Kit: `requestPayment` raises the Huawei Pay checkout screen.
4. User pays on the Huawei Pay client; the Huawei Pay client receives the result.
5. Huawei Pay client shows the result page; on close it returns the payment state
   to the merchant client.
6. Huawei Pay server -> merchant server: callback with the payment result.
7. Merchant server: SM2 signature verification of the callback.

The critical property of this flow is that the loop closes on the server (steps 6
and 7), not on the client (step 5). Step 5 exists to update the UI, not to
release goods.

## Implementation steps

1. **Register the merchant and bind the AppID.** In AppGallery Connect activate
   Payment Kit under **Earn > Payment Kit > PetalPayMerchant (non-virtual
   category)** and associate the merchant ID. The non-virtual category is what
   makes HW-19-0002 matter: virtual goods cannot be sold through this path.
2. **Configure certificates** for request signing and for SM2 verification of the
   result callback; the Huawei Pay certificate is the public key used to verify.
3. **Create the order on the merchant server** when the client asks for checkout.
4. **Call the pre-ordering API** matching the merchant model - direct-merchant
   pre-order, or platform-merchant/service-provider pre-order - to obtain
   `prepayId`. The request body and the `PayMercAuth` request-header object must
   be sorted, concatenated and signed per the official signing rules.
5. **Return `orderStr` to the client**, containing `app_id`, `merc_no`,
   `prepay_id`, `timestamp`, `noncestr`, `sign` and `auth_id`.
6. **Raise the checkout screen** with `paymentService.requestPayment(context,
   orderStr)`, obtaining the `UIAbilityContext` from
   `this.getUIContext().getHostContext()`. Resolution means the API request was
   properly responded to; rejection carries `error.code` from the Payment Kit
   error-code list.
7. **Update the UI only** in `.then()`. Do not unlock the purchase there
   (HW-19-0003).
8. **Implement the result callback endpoint** whose URL you passed as
   `callbackUrl` in the pre-order request, and verify every notification with
   SM2 before trusting it: use the complete notification, sort and concatenate
   the returned data with the `sign` field excluded from the verified content,
   and verify against the Huawei Pay certificate.
9. **Release the goods on the server**, driven by the verified callback or by the
   order query API.
10. **Test in the sandbox environment** before going live.

## Verified snippets

The document has no sample project, so no ZIP-sourced snippet exists for this
card. The snippet below is the document's own, reproduced only to show which
overload it uses; it matches the two-argument
`requestPayment(context: common.UIAbilityContext, orderStr: string):
Promise<void>` form documented in
`documentation/harmonyos-guides/07_application-services/payment-partner-combined.md:49`.

`huawei_industry_tree/19_common_technical_solutions/docs/06_practice-common-huawei-pay-v1.md` (document, not a sample):

```ts
import { BusinessError } from '@kit.BasicServicesKit';
import { paymentService } from '@kit.PaymentKit';
import { common } from '@kit.AbilityKit';

@Entry
@Component
struct Index {
  context: common.UIAbilityContext = this.getUIContext().getHostContext() as common.UIAbilityContext;

  requestPaymentPromise() {
    // use your own orderStr
    const orderStr = '{"app_id":"***","merc_no":"***","prepay_id":"xxx","timestamp":"1680259863114","noncestr":"1487b8a60ed9f9ecc0ba759fbec23f4f","sign":"****","auth_id":"***"}';
    paymentService.requestPayment(this.context, orderStr)
      .then(() => {
        // the checkout screen returned - update the UI only, do NOT release goods here (HW-19-0003)
        console.info('succeeded in paying');
      })
      .catch((error: BusinessError) => {
        console.error(`failed to pay, error.code: ${error.code}, error.message: ${error.message}`);
      });
  }

  build() {
    Column() {
      Button('requestPaymentPromise')
        .type(ButtonType.Capsule)
        .width('50%')
        .margin(20)
        .onClick(() => {
          this.requestPaymentPromise();
        })
    }
    .width('100%')
    .height('100%')
  }
}
```

Note the overload choice. If you need the **unified** checkout (Payment Kit plus
third-party payment methods) rather than the Huawei Pay checkout, the API is the
three-argument form documented in
`documentation/harmonyos-guides/07_application-services/payment-common-pay-mix.md:119`:
`requestPayment(context: common.UIAbilityContext, orderStr: string, payload:
string): Promise<PayResult>`, resolved as
`.then((payResult: paymentService.PayResult) => ...)`. The two are different
integrations; do not mix the argument list of one with the result type of the
other.

## Permissions & config

The document specifies no permissions and no `module.json5` entries, and none of
the Payment Kit checkout APIs require a declared permission. The configuration
this feature does require lives outside the app package:

- AppGallery Connect: Payment Kit activated, merchant registered under
  **PetalPayMerchant (non-virtual category)**, merchant ID bound to the AppID.
- Merchant server: pre-ordering request signing keys, and the Huawei Pay
  certificate for SM2 verification of the result callback.
- `callbackUrl` supplied in the pre-order request; it is the endpoint the Huawei
  Pay server calls in step 6.

## Constraints

- **Non-virtual goods only.** Payment Kit merchants activate under the non-virtual
  PetalPayMerchant category; in-app virtual goods belong to IAP Kit, whose
  product types are consumables, non-consumables, auto-renewable and
  non-renewable subscriptions. (HW-19-0002)
- **Capability availability depends on the merchant model.** Combined payment is
  platform-merchant only; pay-and-sign and sign-and-withhold are direct-merchant
  and service-provider only. Pick the model before designing the flow.
- **If the user is not signed in**, the system automatically launches the HUAWEI
  ID sign-in screen first.
- **The client result is not the payment result.** The official guide is explicit
  that the server notification or the query API must be used instead.
  (HW-19-0003)
- **SM2 verification has exact rules**: verify the complete notification, sort and
  concatenate the returned data, exclude the `sign` field from the verified
  content, and use the Huawei Pay certificate as the public key.
- The document states no device-type or API-level restriction; Payment Kit
  availability is a region and merchant-registration matter, not an SDK version
  matter.

## Pitfalls

- **The overview says Huawei Pay covers 虚拟娱乐 ("virtual entertainment")
  scenarios, which is incorrect for in-app virtual goods.** The same document
  opens by scoping payment to 实体商品或服务 ("physical goods or services"), and
  Payment Kit merchants register under the non-virtual PetalPayMerchant category;
  virtual goods must go through IAP Kit. Read the scenario list as merchant
  industries, and route any in-app virtual product to IAP Kit. (HW-19-0002)
- **The snippet marks the promise resolution as "pay success" with no caveat,
  which is incorrect as a release-the-goods signal.** Use the server-side result
  callback notification (SM2-verified) or the order query API as the
  authoritative result; the client promise only tells you the checkout screen
  returned. (HW-19-0003)
- **Do not copy the two-argument call into a unified-checkout integration.** The
  unified checkout takes a third `payload` argument and resolves with a
  `PayResult`; the Huawei Pay checkout shown here takes two arguments and
  resolves with `void`.
- **The `orderStr` in the snippet is a masked placeholder** (`"app_id":"***"`,
  `"sign":"****"`). It is not a credential and must be built per request on the
  server; never hardcode a real `orderStr` or signing key in the client.

## References

- `documentation/harmonyos-guides/07_application-services/payment-common-pay-mix.md` -
  the three-argument `requestPayment` signature, the unified-checkout sample, the
  pre-ordering procedure, the SM2 verification rules, and the NOTE that the
  client-returned result must not be used as the payment result.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/payment-common-pay-mix
- `documentation/harmonyos-guides/07_application-services/payment-partner-combined.md` -
  the two-argument `requestPayment(context, orderStr): Promise<void>` and
  `AsyncCallback` overloads used by the document's snippet.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/payment-partner-combined
- `documentation/harmonyos-guides/07_application-services/payment-config-agc.md` and
  `payment-binding-appid-to-merc.md` - PetalPayMerchant (non-virtual category)
  activation and merchant-ID binding.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/payment-config-agc
- `documentation/harmonyos-guides/07_application-services/iap-introduction.md` -
  IAP Kit as the kit for virtual goods, with the four product types.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/iap-introduction
- `documentation/harmonyos-guides/07_application-services/payment-kit-guide.md` -
  index of the Payment Kit guides (preparations, certificates, sandbox test,
  access specifications).
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/payment-kit-guide
- https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-common-huawei-pay-v1-0000002004896593
