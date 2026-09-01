# Module types: HAP, HAR, HSP

Which package type each module in [layered-architecture.md](layered-architecture.md)
becomes. Set `"type"` in the module's `src/main/module.json5`.

Source: the official HAP, HAR and in-app HSP guides, linked under
[See also](#see-also).

## Decision

```
Must it be launchable / installable on its own?
  yes -> HAP    (entry if it owns the app icon, otherwise feature)
  no  -> Is it referenced by more than one HAP or HSP?
           no  -> HAR
           yes -> is package size or startup time a concern?
                    yes -> HSP    (one copy, loaded on demand)
                    no  -> HAR    (simpler, no consistency verification)
```

Default for the common-capability and basic-feature tiers is **HAR**. Reach for
HSP only when duplication actually hurts.

## Comparison

| | HAP | HAR | HSP |
|---|---|---|---|
| `type` value | `entry` / `feature` | `har` | `shared` |
| Installs and runs alone | yes | no | no, ships with its HAP |
| Code sharing | - | static, copied into each dependent at build | dynamic, one copy, loaded on demand |
| Owns the app icon | entry only | no | no |
| `pages` tag in module.json5 | yes | **no** | yes |
| Can declare UIAbility | yes | since API 14 | since API 14, not an entry one |
| Can declare ExtensionAbility | yes | since API 18, not an entry one | since API 18, not an entry one |
| Can read `AppScope/` resources | yes | **no** | yes |
| Releasable to OHPM | no | yes | in-app only |

## Constraints that bite

- **A HAR cannot declare `pages`.** It can still contain pages - route to them
  through Navigation, not through the pages profile. This is why
  [navigation.md](navigation.md) matters for any multi-module app.
- **A HAR cannot reference `AppScope/`.** AppScope content is not packaged into
  the HAR at build time. Put anything a HAR needs in the HAR's own resources.
- **No cyclic dependency, no dependency transfer** - for both HAR and HSP. If
  HAR-A depends on HAR-B and HAR-B depends on HAR-C, A cannot use C directly.
  Declare the dependency you actually use.
- **Multiple dependents on one HAR duplicate it.** "When multiple HAPs or HSPs
  reference the same HAR, the application package may contain multiple copies of
  code and resource files." That duplication is the reason HSP exists.
- **A HAR depending on an HSP becomes app-private.** HSPs are intra-application
  only, so such a HAR must not be published to a second- or third-party
  repository; the build will fail for consumers.
- **Permissions merge automatically.** "When a HAP references a HAR, the system
  automatically combines their permission configurations during compilation and
  build." Do not repeat the same permission in both - see
  [permissions.md](permissions.md).

## Declaring it

Entry HAP - `product/phone/src/main/module.json5`:

```json5
{
  "module": {
    "name": "phone",
    "type": "entry",
    "mainElement": "EntryAbility",
    "deviceTypes": ["phone", "tablet", "2in1"],
    "pages": "$profile:main_pages",
    "abilities": [{ "name": "EntryAbility", "exported": true,
      "skills": [{ "entities": ["entity.system.home"],
                   "actions": ["action.system.home"] }] }]
  }
}
```

HAR - `common/src/main/module.json5`, and the same for every feature module:

```json5
{
  "module": {
    "name": "common",
    "type": "har",
    "deviceTypes": ["default", "tablet", "2in1"]
  }
}
```

Both from `NewsSolutionDemo.zip`. Note how little a HAR needs: name, type,
device types. No `pages`, no `mainElement`.

Wire the dependency in the consumer's `oh-package.json5`:

```json5
"dependencies": { "@ohos/common": "file:../../common" }
```

and register the module in the root `build-profile.json5` `modules` list.

## See also

- [layered-architecture.md](layered-architecture.md) - which tier gets which type
- [project-skeleton/](project-skeleton/) - a working set of these files
- `documentation/harmonyos-guides/01_getting-started/hap-package.md`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/hap-package
- `documentation/harmonyos-guides/01_getting-started/har-package.md`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/har-package
- `documentation/harmonyos-guides/01_getting-started/in-app-hsp.md`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/in-app-hsp
