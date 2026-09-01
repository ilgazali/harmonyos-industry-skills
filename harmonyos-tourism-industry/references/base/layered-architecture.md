# Layered architecture

The package layout every HQ industry sample follows. Decide this before
creating a single module; retrofitting it costs a rewrite.

Source: HQ's layered modularisation practice, cross-checked against the official
HAP packaging guide. Both are linked under [See also](#see-also); the local
paths there are provenance from the review repository, not files this skill
needs.
https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-common-app-layered-v1-0000001916033058

## The three tiers

| Tier | Chinese name | Contents | Package type |
|---|---|---|---|
| Product customisation | 产品定制层 | Per-device business logic: UI, resources, configuration | entry-type HAP |
| Basic feature | 基础特性层 | Application features, each high-cohesion and low-coupling | feature HAP if it must deploy independently; otherwise HAR or HSP |
| Common capability | 公共能力层 | What features depend on: UI components, utilities, network, storage | HAR or HSP only |

The stated goal is 一次开发，多端部署 - develop once, deploy to multiple
devices. One codebase whose layering lets each device get the right subset of
packages.

Three tiers are a floor, not a ceiling. Subdivide inside a tier when the app
warrants it.

## The one rule that matters

**Dependencies point one way only:**

```
product customisation  ->  basic feature  ->  common capability
```

No reverse edges. No cycles between siblings. Verify the module graph is
acyclic before first release - the layering only pays off if the direction is
enforced.

## What this looks like on disk

From `NewsSolutionDemo.zip` (`11_news_reading`), the clearest instance of the
doctrine in the corpus:

```
NewsSolutionDemo/
  AppScope/app.json5           bundle name, version, icon - one per app
  build-profile.json5          the module list and SDK version
  oh-package.json5             third-party dependencies
  product/
    phone/                     entry HAP - the only launchable module
  features/
    news/  video/  live/  personal/  service/     HARs
  common/                      HAR - depended on by everything above
```

`product/phone/oh-package.json5` declares the direction explicitly:

```json5
{
  "name": "phone",
  "dependencies": {
    "@ohos/common": "file:../../common",
    "@ohos/live": "file:../../features/live",
    "@ohos/news": "file:../../features/news",
    "@ohos/personal": "file:../../features/personal",
    "@ohos/video": "file:../../features/video"
  }
}
```

`common/oh-package.json5` has `"dependencies": {}` - it is a leaf, as the tier
definition requires.

Root `build-profile.json5` registers every module and marks which one ships as
the product:

```json5
"modules": [
  { "name": "phone", "srcPath": "./product/phone",
    "targets": [{ "name": "default", "applyToProducts": ["default"] }] },
  { "name": "news",   "srcPath": "./features/news" },
  { "name": "common", "srcPath": "./common" }
]
```

Naming varies across the corpus - `common/` vs `commons/`, `features/` vs
`components/`, `product/` vs `entry/` - but the three tiers and the one-way
direction do not.

## Split or share, per module

Decide packaging per module, based on how much the behaviour actually differs
across devices:

- **Entry shared packaging** - one Default entry HAP distributed to several
  devices. Use when the app has a desktop icon and tablet/PC differ little.
- **Entry split packaging** - a separate entry HAP per device type. Use when
  module behaviour differs substantially, or layouts diverge strongly at the
  same breakpoint.

The source sentence describing split packaging is truncated in the published
document (`HW-19-0001`); read 针对不同 as 针对不同**设备**, per device type.

## Constraints

- At most **one entry HAP per device type**. An app package contains either one
  entry HAP, or one entry HAP plus one or more feature HAPs.
- **A single HAP is the recommended default.** The official guide: "If your
  application only uses the UIAbility (that is, no ExtensionAbility is
  required), a single HAP (entry HAP) is recommended."
- **Multi-HAP has a real cost.** "Multiple HAPs reference the same library file,
  which may cause repeated packaging." Choose feature HAPs only when
  independent deployment is genuinely required.
- **Common utilities in a HAP is a dead end.** A HAP cannot be consumed as a
  library dependency by another module. The common-capability tier is
  restricted to HAR or HSP.

## Do not over-read

`单Entry HAP包多HAR包` (one entry HAP, many HARs) is what the document reports
as *common in practice* for single-device apps with no dynamic-deployment
requirement. It is not a prescribed shape for every app.

## See also

- [module-types.md](module-types.md) - choosing HAP vs HAR vs HSP
- [project-skeleton/](project-skeleton/) - the layout above as a starting point
- `documentation/harmonyos-guides/01_getting-started/hap-package.md`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/hap-package
- `documentation/harmonyos-guides/01_getting-started/har-package.md`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/har-package
- `documentation/harmonyos-guides/01_getting-started/in-app-hsp.md`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/in-app-hsp
