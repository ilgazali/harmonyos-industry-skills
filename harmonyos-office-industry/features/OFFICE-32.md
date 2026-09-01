---
id: OFFICE-32
title: Industry FAQ placeholder for the office industry - an empty redirect, and what belongs in its place
industry: 05_office
doc: huawei_industry_tree/05_office/docs/32_practice-office-app-architecture-v1_2.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-office-app-architecture-v1_2-0000002263504048
sample: none (architecture / FAQ document)
kits: []
apis: []
permissions: []
min_api: n/a
modules: []
findings: [HW-05-0182, HW-05-0183]
status: verified-with-fixes
---

## When to use

Load this card when you reach the end of the office industry tree looking for
**行业常见问题** ("industry FAQs") and want to know whether there is anything
there.

There is not. The source document is a single redirect line, and the URL it
redirects to is the same one all nineteen industries redirect to, so nothing
behind it is office-specific (HW-05-0182). This card therefore serves the
purpose the missing page was supposed to serve: it collects the questions that
actually recurred across the 31 reviewed office documents, each answered from
evidence already verified in a scenario card.

## Feature checklist

As a document, this page has one job and does not do it. It should:

- Answer the questions that recur across the office industry's scenarios.
- Or redirect to a destination that is scoped to this industry.
- Sit as the third part of the architecture series, after the framework document
  (OFFICE-01) and the scenario index (OFFICE-02).

Verified: the file is 11 lines including frontmatter; the body is one sentence.
No section of the standard scenario structure - 场景介绍, 实现思路, 约束与限制,
工程目录, 参考文档 - is present, and there is no sample project, so only axes E
(documentation) and F (consistency) apply.

## Architecture

Where this page sits:

```
architecture-guides / office
  practice-office-app-architecture-v1        -> OFFICE-01  综合办公应用案例   (349 lines, has a ZIP)
  practice-office-app-architecture-v1-5_1    -> OFFICE-02  关键场景示例       (index of 29 scenarios)
  practice-office-app-architecture-v1_2      -> OFFICE-32  行业常见问题       (this page - empty)
```

The middle slug is the anomaly: eleven other industries name their scenario
index `..._v1_1`, so their series reads `v1 -> v1_1 -> v1_2`. In `05_office` the
index sits behind `v1-5_1` and sorts outside the series, which leaves `v1_2` -
this empty page - looking like part two (HW-05-0183).

The stub is identical in all nineteen industries:

```
01_auto/docs/08_practice-auto-app-architecture-v1_2.md
02_convenient_life/docs/31_practice-convenient-life-app-architecture-v1_2.md
03_sports_health/docs/15_practice-sports-health-app-architecture-v1_2.md
...
05_office/docs/32_practice-office-app-architecture-v1_2.md
...
19_common_technical_solutions/docs/52_practice-common-app-architecture-v1_35.md
```

All nineteen carry byte-identical body text and the same target URL.

## Implementation steps

Nothing to implement. What follows is the FAQ this page should have contained,
drawn from the defects that recurred across `05_office` - each answer points at
the card and finding that established it.

**1. Which office features need a runtime permission, and which do not?**

Fewer than you would expect. The system pickers run in their own process and
return only what the user picked, so they need **no** permission:
`PhotoViewPicker`, `cameraPicker.pick`, `DocumentViewPicker`,
`scanBarcode.startScanForResult` with the default UI, `contact.selectContacts`,
`SaveButton`, and `showAssetsCreationDialog`. What does need `user_grant`:
`READ_CALENDAR` / `WRITE_CALENDAR` (OFFICE-11, OFFICE-27, OFFICE-31),
`CAMERA` when you drive the camera session yourself rather than through the
picker (OFFICE-07), microphone capture (OFFICE-22), and location (OFFICE-03).

**2. What does a correct `user_grant` request look like?**

Three stages, not one. Call `requestPermissionsFromUser`; check **every** entry
of `authResults`; then read `dialogShownResults` - a `false` there means the
system showed no dialog because the user had already answered, and that is the
only case where `requestPermissionOnSetting` is appropriate. On a final refusal,
tell the user. OFFICE-27's `RequestCalendarPermission.ets` is the best
implementation in this industry; OFFICE-31 (HW-05-0174), OFFICE-01 (HW-05-0002),
OFFICE-11 (HW-05-0065) and OFFICE-21 (HW-05-0123) each get part of it wrong.

**3. Why does my listener fire twice after the app is restarted?**

Because it was never unregistered. `windowStage.on('windowStageEvent')`,
`window.on('windowSizeChange')`, `window.on('windowStatusChange')`,
`window.on('avoidAreaChange')` and `harmonyShare.on('knockShare')` all need
their `off`. The lifecycle guide is explicit that `onWindowStageWillDestroy` is
where WindowStage events are unsubscribed. See OFFICE-30 (HW-05-0163) and
OFFICE-31 (HW-05-0175); OFFICE-31's `ScheduleView` and `ShareConfirmDialog` show
the correct pairing in the same project that misses it in the ability.

**4. My `on('windowStageEvent')` states arrive out of order.**

From API 20 the recommended event for order-sensitive logic is
`on('windowStageLifecycleEvent')`; the older event "does not ensure the order of
lifecycle state transitions". OFFICE-30 (HW-05-0162).

**5. Do I have to close a `relationalStore` ResultSet?**

Yes - "If the resultSet is not closed, FD or memory leaks may occur." Close it
in a `finally`, including on early-return paths. OFFICE-28 (HW-05-0149). And
build a **fresh** `RdbPredicates` per query: conditions concatenate with AND by
default, so a reused object silently stops matching anything (HW-05-0150).

**6. Why does my log line print `<private>`?**

The privacy identifier in a `hilog` format string is mandatory and defaults to
`{private}`. Use `%{public}d` / `%{public}s` for diagnostics. Do **not** solve it
by interpolating values into the format string - that bakes them in as plaintext
and is how OFFICE-29 ended up logging an employee's name, staff number and
department against a caller's phone number (HW-05-0157). Also keep the domain
within `[0x0, 0xFFFF]` (OFFICE-07) and never pass an empty format (OFFICE-14).

**7. My promise never resolves / my callback never runs.**

Check for a hand-built promise in an error path. `return new Promise<undefined>(() => undefined)`
never settles, so every awaiting caller hangs with no error - OFFICE-31
(HW-05-0170). Related: an `async` function that does not `await` its own chain
returns before the work is done (HW-05-0171), and an un-awaited `executeSql`
followed immediately by a query on the table it creates is a race
(OFFICE-29, HW-05-0156).

**8. How do I hand a deep link to my app safely?**

Declare the `scheme`/`host` in `skills.uris`, read `want.uri` in **both**
`onCreate` and `onNewWant`, and remember that on a cold start `onCreate` runs
before any page exists - park the payload in `AppStorage` as well as emitting it
on the `eventHub`, and defer anything that needs a `UIContext` (OFFICE-31,
HW-05-0176). Validate every field that arrives over the link, and require user
confirmation before acting on it: an exported ability that writes straight to the
system calendar lets any app on the device plant an event (HW-05-0172).

**9. Which `AvoidAreaType` do I use for the status bar?**

`TYPE_SYSTEM`. `TYPE_CUTOUT` is the notch and is a different rectangle; using it
for the status bar is a recurring error in this industry.

**10. Why does my `@State` change not repaint?**

`@State` observes the object and its first-layer properties only. Mutating an
element of a nested array, or aliasing an exported module-level constant into
`@State`, does not trigger a rebuild. Use `@Observed`/`@ObjectLink`, or `@Trace`
under the V2 decorators.

**11. Does the sample's version block match the project?**

Not always - check `build-profile.json5` yourself. OFFICE-29's document demands
API 20 and the HarmonyOS 6.0.0 SDK while the project sets
`compatibleSdkVersion: "5.0.3(15)"` and declares no `targetSdkVersion` at all
(HW-05-0159).

**12. The project tree in the document does not match the ZIP.**

Also common, and it matters under `caseSensitiveCheck: true`. Verified
mismatches: `entrybackupablility` vs `entrybackupability` (OFFICE-30,
HW-05-0168) and `ability/schedule` vs the shipped `ablility/schedule`, which
both `build-profile.json5` and `oh-package.json5` depend on (OFFICE-31,
HW-05-0179).

## Verified snippets

None. The document contains no code, and there is no sample project, so this
card has nothing to quote from a ZIP. Every technical claim above is carried by
the scenario card and finding id named beside it.

## Permissions & config

None. This document declares nothing and configures nothing.

## Constraints

- **The document has no content**, so it cannot be reviewed on the API, code,
  ArkUI or security axes - only on documentation (E) and consistency (F).
- **The redirect target is shared across all nineteen industries**, so nothing
  behind it can be office-specific.
- **`harmonyos-faqs` is a separate documentation section** from
  `harmonyos-guides` and `harmonyos-references`, and is not part of the
  documentation mirrored under `documentation/` in this repository - the
  destination could not be opened and its contents are therefore not described
  here.
- **This card's FAQ is derived, not official.** Every answer traces to a finding
  verified against a sample ZIP or an official page in a scenario card; none of
  it comes from the document under review, because the document says nothing.

## Pitfalls

- **The document says the industry FAQ has moved, which leaves the section
  empty** - the destination is the single shared URL every other industry
  redirects to, so a developer looking for an office-specific question has
  nowhere to go, and the last node of the office tree carries no reviewable
  statement at all. (HW-05-0182)
- **The office architecture series is numbered `v1` / `v1-5_1` / `v1_2`, which
  is inconsistent** with the `v1` / `v1_1` / `v1_2` scheme used by eleven other
  industries: the scenario index - the most useful page in the industry - sorts
  outside the series, and this empty stub is what a reader walking the URL
  pattern finds as part two. (HW-05-0183)

## References

- The document itself, `huawei_industry_tree/05_office/docs/32_practice-office-app-architecture-v1_2.md`.
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-office-app-architecture-v1_2-0000002263504048
- The two other parts of the series: OFFICE-01
  (`01_practice-office-app-architecture-v1.md`) and OFFICE-02
  (`02_practice-office-app-architecture-v1-5_1.md`), the index of all 29
  scenario documents.
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-office-app-architecture-v1-5_1-0000002267128277
- The eighteen sibling stubs, one per industry, used to establish that the
  redirect is shared - for example
  `huawei_industry_tree/01_auto/docs/08_practice-auto-app-architecture-v1_2.md`.
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-auto-app-architecture-v1_2-0000002297842985
- The scenario cards named against each FAQ answer above: OFFICE-01, OFFICE-03,
  OFFICE-07, OFFICE-11, OFFICE-14, OFFICE-21, OFFICE-22, OFFICE-27 through
  OFFICE-31.
