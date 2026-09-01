---
id: SPORT-10
title: Knockout bracket - Polyline connectors over an index-addressed match array
industry: 03_sports_health
doc: huawei_industry_tree/03_sports_health/docs/10_tournament_advancement_chart.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/tournament_advancement_chart-0000002381782357
sample: huawei_industry_tree/03_sports_health/downloads/KnockoutMatchList.zip
kits: ["@kit.ArkUI", "@kit.CryptoArchitectureKit"]
apis: [Polyline, points, strokeWidth, stroke, Tabs, TabContent, Row, Column, ForEach, "@Prop", "@State", "cryptoFramework.createRandom"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-03-0039, HW-03-0040, HW-03-0041, HW-03-0055]
status: verified-with-fixes
---

## When to use

Load this card for a **knockout bracket** - the fan-in diagram where winners
advance from sixteen to eight to four to one. Football here, and the document
extends it to basketball, tennis and esports.

Two things make it worth reading:

- **`Polyline` as the connector.** Each advancement line is a four-point
  polyline in its own box, so the bracket's tree structure is drawn without a
  `Canvas`, without measuring, and without absolute positioning.
- **A flat array addressed by arithmetic.** Fifteen match groups live in one
  array, and a team's index plus the round number computes its slot - so
  filling the bracket from a flat list of fixtures is one formula rather than
  a tree walk.

The second idea is the transferable one, and it is where all three findings
are: index arithmetic is only as safe as the values fed into it.

## Feature checklist

- A full sixteen-team knockout bracket: round of 16, quarter-finals,
  semi-finals, final.
- Both completed and in-progress tournaments render.
- Each tie shows both teams' crests, names and aggregate score.
- Extra time and penalty results are shown as an additional line where they
  occurred.
- Winners are highlighted and carried into the next round automatically.
- Connector lines join each pair of ties to the tie they feed.

## Architecture

One `entry` module, three pages, one model, one constants file.

```
entry/src/main/ets
├── constants/CommonConstants.ets    POWER_OF_ROUND and COUNT_OF_ROUND
├── entryability/EntryAbility.ets
├── entrybackupability/
├── model/SourceDataModel.ets        Competitor, MatchGroup, SourceData, SOURCE_DATA1/2
└── pages
    ├── Index.ets                    the Tabs host
    ├── KnockoutMatchView.ets        the bracket: fills the array, lays out the rounds
    └── SingleRoundMatchView.ets     one reusable tie group, with its Polyline connectors
```

The documented tree describes `SourceDataModel.ets` as a font-loading class
(`HW-03-0041`).

**The bracket is a flat array of 15 groups, addressed by arithmetic:**

```
index:   0  1  2  3  4  5  6  7 │  8  9 10 11 │ 12 13 │ 14
round:   ── round of 16 (8) ──  │ quarters(4) │ semis │ final

COUNT_OF_ROUND = [0, 8, 12, 14]     // groups completed before each round
POWER_OF_ROUND = [0, 2, 4, 8, 16]   // teams per group at each round

matchesIndex = ~~((teamIndex - 1) / POWER_OF_ROUND[round]) + COUNT_OF_ROUND[round - 1]
```

So team 5 in round 1 lands at `~~(4/2) + 0 = 2`; the same team in round 2
lands at `~~(4/4) + 8 = 9`. **One formula places any fixture from any round**,
which is what lets the loader take an unordered list of source records and
fill the bracket in a single pass.

The same formula, applied with `round + 1`, is what promotes a winner into the
next group - so advancement needs no parent pointers.

**Every tie group is the same component.** `SingleRoundMatchView` renders one
pair plus its connectors, parameterised by widths, so the four rounds differ
only in the numbers passed to it.

## Implementation steps

1. **Pre-allocate all 15 groups** so every slot exists before any record is
   read.
2. **Compute the slot from the team index and the round** with integer
   division.
3. **Guard the record before the arithmetic** - a missing opponent must be
   skipped, not truncated to slot 0 (`HW-03-0040`).
4. **Accumulate the aggregate** across both legs, folding in penalties.
5. **Decide the winner once**, and handle the tie explicitly
   (`HW-03-0039`).
6. **Promote the winner** with the same formula at `round + 1`.
7. **Draw connectors as `Polyline`s** inside the group's own box, mirrored for
   the two halves of the bracket.

## Verified snippets

All snippets are from `KnockoutMatchList.zip`. Corrected forms are marked.

**Placing a fixture — `entry/src/main/ets/pages/KnockoutMatchView.ets`** (corrected, see `HW-03-0040`)

```typescript
// constants
static readonly POWER_OF_ROUND = [0, 2, 4, 8, 16];
static readonly COUNT_OF_ROUND = [0, 8, 12, 14];  // groups that exist before this round starts

// 15 empty groups: 8 for the round of 16, 4 quarters, 2 semis, 1 final
this.matches = [];
for (let i = 0; i < 15; i++) {
  this.matches.push(new MatchGroup());
}

for (let i = 0; i < sourceData.length; i++) {
  const curSourceData = sourceData[i];
  if (curSourceData.opponentA === undefined) {
    continue;                          // FIX: the sample lets undefined reach the arithmetic
  }
  const idx = curSourceData.opponentA.teamIndex;
  const round = curSourceData.round;
  // ArkTS division is floating point; ~~ truncates to get integer division
  const matchesIndex =
    ~~((idx - 1) / CommonConstants.POWER_OF_ROUND[round]) + CommonConstants.COUNT_OF_ROUND[round - 1];

  const opponents = [curSourceData.opponentA, curSourceData.opponentB];
  for (let j = 0; j < 2; j++) {
    this.matches[matchesIndex].competitors[j].refresh(
      opponents[j].teamName, opponents[j].teamIcon, opponents[j].score,
      opponents[j].additionalScore, opponents[j].teamIndex);
  }
  this.matches[matchesIndex].setScore(
    opponents[0].score.toString() + '-' + opponents[1].score.toString());
}
```

**`~~` is integer division in ArkTS**, and the comment in the sample says why
it is there: `/` is floating point, so `~~` truncates. It is worth knowing
that `~~` also turns `NaN` into `0` rather than propagating it, which is
exactly what makes the missing guard dangerous.

**Pre-allocating all fifteen groups first** is what lets the loader write any
round in any order - the promotion step writes into a group whose fixture may
not have been read yet.

**Aggregating and advancing — same file** (corrected, see `HW-03-0039`)

```typescript
// model: the aggregate accumulates across legs and folds in penalties
public refresh(name: string | Resource, icon: Resource, score: number,
  additionalScore: number, index: number) {
  // ...
  this.totalScore += (score + additionalScore);
}

// the tie is decided once both legs are in - or immediately, for the single-legged final
if (this.matches[matchesIndex].scores.length === 2 || curSourceData.round === 4) {
  const c1 = this.matches[matchesIndex].competitors[0];
  const c2 = this.matches[matchesIndex].competitors[1];

  // FIX: the sample sets two flags with strict > and then reads winner from a ternary,
  //      so an equal aggregate marks both as losers and advances c2 regardless
  if (c1.totalScore === c2.totalScore) {
    return;                              // no winner yet: leave the next slot empty
  }
  const winner = c1.totalScore > c2.totalScore ? c1 : c2;
  c1.setWinner(winner === c1);
  c2.setWinner(winner === c2);

  if (curSourceData.round < 4) {
    const nextRound = curSourceData.round + 1;
    const nextMatchIndex = ~~((winner.index - 1) / CommonConstants.POWER_OF_ROUND[nextRound]) +
      CommonConstants.COUNT_OF_ROUND[nextRound - 1];
    const nextCompetitorIndex = matchesIndex % 2;      // even groups feed the top slot, odd the bottom
    this.matches[nextMatchIndex].competitors[nextCompetitorIndex].refresh(
      winner.teamName, winner.teamIcon, 0, 0, winner.index);
  }
}
```

**`scores.length === 2 || round === 4` is the completeness test.** Two legs
recorded means the tie is over; the final is single-legged, so the round
number is the exception. **`matchesIndex % 2` decides which half of the next
group the winner occupies** - group 0 and 1 both feed group 8, one into the
top slot and one into the bottom - which is the piece that makes the flat
array behave like a tree.

Promoting with `refresh(..., 0, 0, ...)` seeds the next round at zero, and
because `totalScore` accumulates, a group's competitor must be a fresh object
per group - which the pre-allocation guarantees.

**The connectors — `entry/src/main/ets/pages/SingleRoundMatchView.ets`** (as shipped)

```typescript
Row() {
  // left branch: up from the bottom-left, across, then up to the join
  Polyline({ width: this.lineWidth, height: 28 })
    .points([[0, 28], [0, 14], [this.lineWidth, 14], [this.lineWidth, 0]])
    .strokeWidth(3)
  // right branch: the mirror image
  Polyline({ width: this.lineWidth, height: 28 })
    .points([[this.lineWidth, 28], [this.lineWidth, 14], [0, 14], [0, 0]])
    .strokeWidth(3)
}
```

**Four points draw the whole bracket connector**: up the outer edge, in along
the midline, then up to the parent. The second polyline is the first with its
x coordinates mirrored, so one shape serves both sides.

The polyline's own `width` and `height` define its coordinate space, so the
points are expressed relative to the box rather than to the page - which is
why the same component works at every round with only `lineWidth` changing.
`lineWidth` is passed per round (30 for the round of 16, 46 for the quarters)
because the horizontal gap between rounds grows as the bracket narrows.

**Choosing a dataset — `KnockoutMatchView.ets`** (as shipped)

```typescript
import { cryptoFramework } from '@kit.CryptoArchitectureKit';

const rand = cryptoFramework.createRandom();
const promiseGenerateRand = rand.generateRandomSync(1);
const index = promiseGenerateRand.data[0];
const sourceData: Array<SourceData> = index % 2 === 1 ? SOURCE_DATA1 : SOURCE_DATA2;
```

Two datasets - one finished tournament and one in progress - picked at random
so the sample demonstrates both states. A cryptographic RNG is heavier than
this needs, but it is the platform's supported random source.

## Permissions & config

**None.** The sample declares no `requestPermissions` - it is a drawing over
static data.

No routing configuration; `Index.ets` hosts the bracket in `Tabs`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **The bracket is fixed at sixteen teams.** `COUNT_OF_ROUND` and
  `POWER_OF_ROUND` are hardcoded for four rounds and fifteen groups, and the
  layout in `KnockoutMatchView.build` slices the array at literal offsets
  (`0..4`, `8..10`) with literal widths and heights. A 32-team draw means new
  constants and a new layout.
- **Ties are assumed to be resolved by the aggregate**, penalties included.
  Away goals, seeding and other tie-breaks have nowhere to live in the model.
- The round-of-16 groups render four to a screen with fixed sizes
  (338 by 76 or 108 vp), so the bracket does not adapt to screen width.
- Source data is two static arrays; there is no fixture service.

## Pitfalls

- **`HW-03-0039` — an equal aggregate marks both competitors as losers,** and
  the advancement ternary then promotes the second one anyway, so the chart
  and the progression disagree.
- **`HW-03-0040` — the slot is computed from `opponentA?.teamIndex`,** and
  `undefined` becomes `NaN` becomes `0` through `~~`, so a record with a
  missing opponent silently overwrites the first round-of-16 group.
- **`HW-03-0041` — the documented tree calls `SourceDataModel.ets` a
  font-loading entity class,** although it is the match data model the whole
  document is about.

## References

- `documentation/harmonyos-references/02_application-framework/ts-drawing-components-polyline.md` - `Polyline`, `points`, `strokeWidth`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-drawing-components-polyline
- `documentation/harmonyos-references/02_application-framework/ts-container-tabs.md` - `Tabs` and `TabContent`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-tabs
- `documentation/harmonyos-references/02_application-framework/ts-container-row.md`, `ts-container-column.md` - the layout primitives
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-row
- `documentation/harmonyos-references/02_application-framework/ts-rendering-control-foreach.md` - `ForEach`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-rendering-control-foreach
- `documentation/harmonyos-references/03_system/js-apis-cryptoFramework.md` - `createRandom` and `generateRandomSync`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-cryptoframework
- `SPORT-06` - the scorer that produces the results this bracket consumes
- `SPORT-05`, `SPORT-07` - the industry's other two data visualisations
