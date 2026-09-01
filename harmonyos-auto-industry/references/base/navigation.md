# Navigation

How the app moves between screens. Source: HQ's navigation design practice,
cross-checked against the official Navigation introduction. Both are linked
under [See also](#see-also).
https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-common-app-navigation-v1-0000001946960441

## The shape

- **One UIAbility instance.** One UIAbility maps to one entry in the task
  centre, so extra instances mean extra task-centre entries. Add a second only
  when you deliberately want that.
- **Many ArkUI pages inside it**, not one ability per screen.
- **Navigate through the routing layer**, never by starting abilities.
- **Use `Navigation`, not `@ohos.router`.**
- **Register routes dynamically**, so opening one page does not construct every
  module's UI components.
- **Keep route registration out of the feature modules**, so they never import
  each other.

Concretely: `EntryAbility` loads one page; that page owns a single
`NavPathStack` and provides it to the tree; every feature module exports
`NavDestination`-rooted components; a route table maps a name string to a
builder; navigation is `pushPathByName(name, param)` and `pop()`. Because the
mapping resolves by name at runtime, no feature module needs a compile-time
reference to another.

This is also what makes a multi-module app work at all: a HAR cannot declare
`pages` (see [module-types.md](module-types.md)), so Navigation is the only way
to reach a screen that lives in a feature HAR.

## Navigation vs Router

| | Navigation | Router (`@ohos.router`) |
|---|---|---|
| Page unit | content of a `NavDestination` | component decorated `@Entry` |
| Position in tree | container component, stackable and nestable | beneath the page-stack node |
| Recommended | **yes**, long-term direction | supported, not the direction |

The official guide: Navigation offers "enhanced multi-device adaptation
capabilities, more flexible page stack management, and richer animation effects
and lifecycle support. Therefore, to deliver a better user experience,
prioritize the **Navigation** component."

Router cannot do these at all:

- one-time development for multi-device deployment (Auto mode single/double
  column adaptation)
- obtaining parameters of a specific page
- obtaining the page stack object
- route interception (`setInterception`)
- removing specific routes (`removeByIndexes`, `removeByName`)
- moving within the stack (`moveToTop`, `moveIndexToTop`)
- shared-element animation via `geometryTransition`
- immersive pages without extra window configuration

If the product needs any of these, Navigation is a requirement, not a
preference.

Router also restricts parameter shape: parameters pass as an object, but
"method variables not supported within the object". Navigation has no such
restriction.

## Steps

1. Plan one UIAbility.
2. Split the UI into ArkUI pages inside it.
3. Wrap the home content in a `Navigation` component; create exactly one
   `NavPathStack` for the app.
4. Root every routed screen in a `NavDestination`, not in an `@Entry` component.
5. Register routes dynamically - bind a builder to `Navigation` that resolves a
   route name to a component at navigation time. This is the step that fixes
   both slow startup and cross-module cycles.
6. Navigate by name, pass parameters as objects.
7. For confirm-before-leave and similar, use `setInterception` on the stack.
   Router's only equivalent is `showAlertBeforeBackPage`.

Under Navigation there is no router table in `module.json5`; the mapping lives
in the page builder.

## Do not over-read

- **"Navigation is recommended" is not "Router is deprecated."** The Router
  class reference carries no deprecation notice. Do not flag existing Router
  code as deprecated API, and do not refactor working Router code on that basis.
- **"Single UIAbility" is a design recommendation, not a platform limit.** The
  reason is task-centre behaviour, not an inability to create more.

## See also

- `COMMON-50` in `harmonyos-common-industry` - Navigation route continuation,
  with code verified against a sample project
- `documentation/harmonyos-guides/03_application-framework/arkts-navigation-introduction.md`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-navigation-introduction
- `documentation/harmonyos-guides/03_application-framework/arkts-navigation-navigation.md`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-navigation-navigation
