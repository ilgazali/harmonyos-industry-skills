# Project skeleton

A three-tier HarmonyOS project in its smallest working form. Copy it, rename
the bundle, add features.

Distilled from `NewsSolutionDemo.zip` (`11_news_reading`), the clearest instance
of the layered doctrine in the corpus. Configuration values are the shipped
ones, not invented.

## Layout

```
build-profile.json5          SDK version, products, module list
oh-package.json5             third-party dependencies
AppScope/app.json5           bundle name, version, icon

product/phone/               product customisation tier - entry HAP
  oh-package.json5             depends on common + every feature
  build-profile.json5
  src/main/module.json5        type: entry, owns EntryAbility and the app icon

features/example/            basic feature tier - HAR
  oh-package.json5             depends on common only
  build-profile.json5
  src/main/module.json5        type: har

common/                      common capability tier - HAR
  oh-package.json5             a leaf, depends on nothing
  build-profile.json5
  src/main/module.json5        type: har
```

Not included, because DevEco generates them: `src/main/ets/`,
`src/main/resources/`, `src/main/module.json5` page profiles,
`hvigorfile.ts`, `obfuscation-rules.txt`, `.gitignore`.

## Adding a feature

1. Copy `features/example/` to `features/<name>/`.
2. Change `name` in its `oh-package.json5` and `src/main/module.json5`.
3. Register it in the root `build-profile.json5` `modules` list.
4. Add `"@ohos/<name>": "file:../../features/<name>"` to
   `product/phone/oh-package.json5`.
5. Export the feature's entry component from its `Index.ets`.

A feature must never appear in another feature's dependencies. If two features
need the same thing, it belongs in `common/`.

## Adding a device target

Copy `product/phone/` to `product/tablet/`, register it, and give it its own
`deviceTypes`. Do this only when behaviour genuinely differs - otherwise widen
`deviceTypes` on the single entry HAP instead. See
[../layered-architecture.md](../layered-architecture.md) on split versus shared
packaging.

## Before you ship

- `bundleName` in `AppScope/app.json5` is still `com.example.myapplication`.
- `requestPermissions` in the entry module is empty on purpose. Add only what
  the code exercises, each with `reason` and `usedScene` - see
  [../permissions.md](../permissions.md).
- `compatibleSdkVersion` and `targetSdkVersion` are `6.1.1(24)`. Lower them only
  if you have checked every API you use against that level.
- Release obfuscation is on in every module. `consumerFiles` is set on the HARs
  so their rules propagate to consumers; the entry HAP has no consumers and so
  does not set it.
