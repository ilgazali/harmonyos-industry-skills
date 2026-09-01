# Pitfalls

> Generated from `features/*.md`. Source industry: `12_jobs`, 5 features.
> Do not edit by hand; regenerate it in the review repository.

Every entry is a confirmed defect in the published HQ documentation or in its sample project. A naive copy of the document reproduces it.

## Systemic - repeated across features (1)

These recur in more than one feature of this industry. Fix them once in your own project template.

### `HW-12-0005` - Systematic: 3 sample projects in this industry ship with release obfuscation explicitly disabled

- Category D, severity low, confidence confirmed
- Features: JOBS-02, JOBS-03, JOBS-04
- Document: `huawei_industry_tree/12_jobs/docs/03_position_sliding_window.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/position_sliding_window-0000002391546225
- Why: These samples are published as templates and are copied wholesale into real products. A release buildOptionSet that sets obfuscation.ruleOptions.enable to false, while still shipping an obfuscation-rules.txt, reads as a deliberate configuration rather than an omission, so a developer copying the module has no signal that release builds are unprotected. ArkTS source names and structure remain readable in the released HAP.
- Fix: Set arkOptions.obfuscation.ruleOptions.enable to true in the release entry of buildOptionSet for every module, and keep the existing obfuscation-rules.txt. HARs should also declare consumerFiles so their rules reach consumers.

## Per feature (4)

### `HW-12-0001` - Authorization flow force-opens the notification settings dialog on every launch once the user has refused

- Category B, severity medium, confidence confirmed
- Features: JOBS-02
- Document: `huawei_industry_tree/12_jobs/docs/02_notification_authorization_popup.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/notification_authorization_popup-0000002386582149
- Why: After a user refuses once, requestEnableNotification rejects immediately on every subsequent launch, so the code falls into the isDialogShown!==true branch and calls openNotificationSettings each time the page appears — an every-launch nag that also double-prompts in the very first session (settings dialog right after the user tapped deny). The catch blocks ignore error codes entirely, so the documented 1600004 'user refused' case is indistinguishable from an API failure; the flag name isDialogShown actually means 'granted'.
- Fix: In requestPermissions() catch, check err.code === 1600004 and persist a 'userRefused' flag (e.g. preferences); only open openNotificationSettings once, or from a user-triggered UI entry point. Rename isDialogShown to isGranted.

### `HW-12-0002` - Card array is not state-observed: deleted cards stay mounted and an always-true condition is used as a re-render hack

- Category B, severity medium, confidence confirmed
- Features: JOBS-03
- Document: `huawei_industry_tree/12_jobs/docs/03_position_sliding_window.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/position_sliding_window-0000002391546225
- Why: The rendered card list and the data array permanently diverge: after all 7 cards are 'deleted' the Stack still holds 7 invisible, gesture-bearing Columns; correctness relies on Card being @ObservedV2 and objects being mutated in place. The tautological condition is dead logic that any reader (or linter) would flag, and it documents the missing observation instead of fixing it.
- Fix: Decorate the array (e.g. @State arr: Card[] or migrate the page to @ComponentV2 with a @Local array), remove updateArrLength and write the conditions as plain `if (this.arr.length > 0)`.

### `HW-12-0003` - Swipe direction can silently keep its previous value when the pan activates without crossing the ±100 offset check

- Category B, severity low, confidence probable
- Features: JOBS-03
- Document: `huawei_industry_tree/12_jobs/docs/03_position_sliding_window.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/position_sliding_window-0000002391546225
- Why: onActionEnd always calls handleSlider(), so a pan whose start offset fell in the dead zone animates the cards in whatever direction the previous gesture set — the card stack can jump the wrong way.
- Fix: Set leftToRight from `event.offsetX > 0` in onActionEnd (or track it in onActionUpdate) instead of thresholding in onActionStart.

### `HW-12-0004` - Invalid fontWeight(40) — outside the documented [100, 900] range, silently falls back to 400

- Category B, severity low, confidence confirmed
- Features: JOBS-04
- Document: `huawei_industry_tree/12_jobs/docs/04_recruitment_and_job_list_display.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/recruitment_and_job_list_display-0000002456685322
- Why: 40 is not a legal font weight; the intended lighter style (likely a typo for 400) is silently replaced by the default, and copying the sample propagates an invalid API value.
- Fix: Change fontWeight(40) to fontWeight(400).

