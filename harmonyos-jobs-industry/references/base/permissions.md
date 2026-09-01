# Permissions

Declaring and requesting them, and the five ways the HQ samples get it wrong.

Source: the official permission guides, linked under [See also](#see-also),
plus 152 permission-related findings across the 19 industry workbooks.

## Four kinds

| Kind | Granted | Notes |
|---|---|---|
| `system_grant`, open | at install | no dialog, no runtime check |
| `user_grant`, open | by the user, at runtime | dialog; needs `reason` and `usedScene` |
| Restricted | ACL | `system_basic` permissions granted to normal apps |
| Enterprise / MDM | by distribution type | `enterprise_normal` or `enterprise_mdm` only |

Only `user_grant` needs the runtime flow below. Requesting a restricted
permission you do not need is a review rejection, not a warning.

## Declaring

In `src/main/module.json5`, under `requestPermissions`:

```json5
"requestPermissions": [
  {
    "name": "ohos.permission.CAMERA",
    "reason": "$string:reason_camera",
    "usedScene": {
      "abilities": ["EntryAbility"],
      "when": "inuse"
    }
  }
]
```

- `reason` is **mandatory** for `user_grant` and `manual_settings`. Reference it
  as a string resource for multilingual support.
- `usedScene` is **mandatory** for the same. `abilities` names real UIAbility or
  ExtensionAbility components; `when` is `inuse` or `always`, never empty.
- `reason` wording is checked at release. Use "Used for video calls." - a
  complete sentence naming the actual scenario. "Obtain the camera permission"
  and "Permission required" are rejected. Under 72 characters recommended,
  256 maximum, never blank.
- If the same permission is requested by several HAPs, the reason must cover
  every scenario, identically, in all of them.

**Declare once per app, not per module.** Permissions declared in the entry
module take effect application-wide and do not need repeating in feature
modules, or vice versa. When a HAP references a HAR, "the system automatically
combines their permission configurations during compilation and build."

## Requesting at runtime

1. Declare in the configuration file.
2. Associate each object needing the permission with it in the UI, so the user
   knows what needs authorisation.
3. Check with `checkAccessToken()`; if not granted, call
   `requestPermissionsFromUser()`.
4. **Inspect the result** and only then proceed.

```typescript
import { abilityAccessCtrl, bundleManager, Permissions } from '@kit.AbilityKit';

async function checkPermissionGrant(permission: Permissions):
    Promise<abilityAccessCtrl.GrantStatus> {
  const atManager = abilityAccessCtrl.createAtManager();
  const bundleInfo = await bundleManager.getBundleInfoForSelf(
    bundleManager.BundleFlag.GET_BUNDLE_INFO_WITH_APPLICATION);
  return atManager.checkAccessToken(bundleInfo.appInfo.accessTokenId, permission);
}
```

Constraints from the guide:

- **Check every time**, before every guarded operation. A granted permission can
  be revoked in Settings; previous authorisation status is not persistent.
- **The dialog appears once.** If the user denies, it will not show again. Guide
  them to Settings, or call `requestPermissionOnSetting()`.
- **Show a rationale in the UI** before requesting.
- **The system dialog cannot be obscured** by other components.
- **Timing in `onWindowStageCreate()`**: wait until `loadContent()` /
  `setUIContent()` completes, or call `requestPermissionsFromUser()` from inside
  it. Otherwise the call happens before the ability is loaded and fails.

## The five recurring defects

Ranked by how often they appear in the workbooks. Every one of these is in
shipped HQ sample code - do not copy the pattern.

**1. `usedScene` names an ability that does not exist (11 findings).** Example
`HW-01-0026`: three permissions declare `usedScene.abilities: ["ShopAbility"]`
in a module that has no such ability. Check the name against `abilities[].name`
in the same `module.json5`.

**2. The grant result is never inspected (9 findings).** Example `HW-01-0021`:
the code requests, ignores `authResults`, and proceeds to use the API even after
the user denied. Always read `authResults` and branch on it.

**3. Declared but never used, or documented but not declared (9 findings).**
Example `HW-18-0004`: nine photography samples declare restricted
`READ_IMAGEVIDEO` / `WRITE_IMAGEVIDEO` although their code is built entirely on
the permission-free `SaveButton` and `PhotoViewPicker` flows and never requests
them - one shared `module.json5` template is the root cause. Delete permissions
your code does not exercise; a restricted permission you cannot justify fails
release review.

**4. No rejection handler, or the call races the grant (7 findings).** Example
`HW-01-0003`: `getCurrentLocation()` has no rejection handler and runs before the
permission request has resolved. Await the grant, then call. Never wrap the
permission path in an empty `catch`.

**5. Multi-permission checks that honour only one (6 findings).** Example
`HW-01-0001`: the helper loops over the permission list but overwrites its result
each iteration, so only the last permission decides - a denied `LOCATION` is
never requested when `APPROXIMATELY_LOCATION` is granted. Example `HW-02-0100`
returns success as soon as *either* location permission is granted. Aggregate
with AND across the whole list.

**6. And one that follows from the flow itself.** `HW-18-0007`:
`requestPermissionsFromUser` rejects the *entire* call when any requested
permission is undeclared in `module.json5`, so one stray entry breaks the grant
for all of them. Keep the runtime array and the declared set in sync.

## Prefer the permission-free path

Several capabilities have flows that need no permission at all, and the corpus
shows samples declaring permissions they then never use:

- Saving to the gallery: `SaveButton` (a security component) plus
  `photoAccessHelper.createAsset` - no `WRITE_IMAGEVIDEO`.
- Picking media: `PhotoViewPicker` - no `READ_IMAGEVIDEO`.

Check whether a security component or picker covers your case before declaring
anything.

## The permissions this corpus actually uses

33 distinct permissions across 443 features. The most common:

| Permission | Features |
|---|---:|
| `ohos.permission.INTERNET` | 58 |
| `ohos.permission.APPROXIMATELY_LOCATION` | 24 |
| `ohos.permission.CAMERA` | 22 |
| `ohos.permission.LOCATION` | 21 |
| `ohos.permission.WRITE_IMAGEVIDEO` | 20 |

Each industry's `references/api-map.md` has the full permission-to-feature
index for that industry.

## See also

- `documentation/harmonyos-guides/04_system/app-permissions.md` - the four kinds
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/app-permissions
- `documentation/harmonyos-guides/04_system/declare-permissions.md` - reason and usedScene rules
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/declare-permissions
- `documentation/harmonyos-guides/04_system/request-user-authorization.md` - the runtime flow and its constraints
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/request-user-authorization
- `documentation/harmonyos-guides/04_system/restricted-permissions.md` - what ACL covers
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/restricted-permissions
- `documentation/harmonyos-guides/04_system/permissions-for-all.md` - open system_grant permissions
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all
