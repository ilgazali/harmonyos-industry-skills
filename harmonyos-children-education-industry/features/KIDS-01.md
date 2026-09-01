---
id: KIDS-01
title: Children's education key-scenario catalog - the fifteen sample scenarios
industry: 08_children_education
doc: huawei_industry_tree/08_children_education/docs/01_practice-kids-app-architecture-v1_1.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-kids-app-architecture-v1_1-0000002270219009
sample: NO ZIP
kits: []
apis: []
permissions: []
min_api: 20
modules: []
findings: []
status: verified
---

## When to use

This is the **index page** of the children's education practice. Load it only
to find which scenario card to open.

The industry has an unusual shape: **there is no architecture document.**
Tourism and sports and health each open with a case study - layers, HAR
modules, a framework project - and only then list their scenarios. This
practice has just two architecture-level pages, and the first of them is this
bare list:

| Document | Card | Role |
| --- | --- | --- |
| 关键场景示例 (key scenario examples) | `KIDS-01` (this card) | the catalog of the fifteen scenario samples |
| 行业常见问题 (industry FAQ) | `KIDS-17` | a redirect notice, no content |

So there is no industry-level module layout, no shared framework code, and no
prescribed layering to inherit. **Each of the fifteen samples is a standalone
single-`entry` project**, and the card for each is self-contained.

Fifteen scenarios makes this the largest scenario set of any industry reviewed
so far, one more than sports and health.

## Feature checklist

The document is a plain list of fifteen links and nothing else - no prose, no
code, no images. All fifteen targets are present in this corpus, all fifteen
zips are downloaded, and `tree.json` carries exactly these fifteen children
under 关键场景示例, so the catalog is complete and matches the sidebar.

| # | Scenario (document title) | Sample | Card |
| --- | --- | --- | --- |
| 1 | 家长权限计算验证弹窗 - arithmetic gate for parent mode | `demo_Caculater.zip` | `KIDS-02` |
| 2 | 儿童练字板 - handwriting practice board | `CanvasDemo.zip` | `KIDS-03` |
| 3 | 防沉迷控制 - screen-time limit and lockout | `ControlUsageTime.zip` | `KIDS-04` |
| 4 | 围棋棋子绘制 - drawing a Go board and stones | `canvasCheckerboard.zip` | `KIDS-05` |
| 5 | 七巧板拼图 - tangram puzzle | `JigsawPuzzle.zip` | `KIDS-06` |
| 6 | 儿童绘画板 - drawing board with colour and brush pickers | `DrawBoard.zip` | `KIDS-07` |
| 7 | 古诗文解析模板 - classical poem with tap-to-annotate | `PoetryTemplate.zip` | `KIDS-08` |
| 8 | Canvas绘制固定图形 - drawing rectangles, circles, triangles | `DrawFixedShape.zip` | `KIDS-09` |
| 9 | 图片华容道 - sliding picture puzzle | `PicturesHuarongRoad.zip` | `KIDS-10` |
| 10 | 设备流量监控 - mobile data usage monitor | `DataMonitor.zip` | `KIDS-11` |
| 11 | 儿童定位与移动轨迹追踪 - child location and movement trail | `MapLocation.zip` | `KIDS-12` |
| 12 | 儿童词语学习卡片 - vocabulary cards with audio | `VocabularyLearningCards.zip` | `KIDS-13` |
| 13 | 距离过大报警器 - distance-from-home alarm | `DistanceAlarm.zip` | `KIDS-14` |
| 14 | 舒尔特方格专注力训练游戏 - Schulte grid focus game | `GridFocusTraining.zip` | `KIDS-15` |
| 15 | 模拟投掷骰子 - simulated dice roll | `DieRolling.zip` | `KIDS-16` |

## Architecture

None of its own. What the catalog does reveal is the **shape of the industry**,
and it is dominated by one component.

**Canvas and touch-drawing - six of the fifteen scenarios.** `KIDS-03`,
`KIDS-05`, `KIDS-06`, `KIDS-07`, `KIDS-09` and, by way of
`backgroundImagePosition` rather than `Canvas` itself, `KIDS-10`. Anything a
child draws, drags or assembles goes through `Canvas` +
`CanvasRenderingContext2D` + `onTouch`, so those three are the industry's
core API surface and are worth reading once rather than fifteen times.

**Location - two scenarios, and they are the parental half of the product.**
`KIDS-12` draws the child's trail on a `MapComponent` with `addTraceOverlay`;
`KIDS-14` registers `geoLocationManager.on('locationChange')` and raises an
alarm past a radius. These are the only two samples in the industry that need
runtime permissions, and the only two that can drain a battery.

**Parental control - three scenarios that have nothing to do with drawing.**
`KIDS-02` gates a settings page behind arithmetic a child cannot do,
`KIDS-04` locks the app after a time budget, `KIDS-11` reports data usage.
Together with the two location samples, these five are the "parent" side of a
children's app; the other ten are the "child" side.

**Two scenarios carry a dependency the rest do not:** `KIDS-13` uses
`AVPlayer` from Media Kit for the word audio, and `KIDS-16` pulls the
third-party `@ohos/gif-drawable@2.1.1` from ohpm to animate the die.

## Implementation steps

None. Open the card for the scenario you need.

## Verified snippets

None - the document contains no code.

## Permissions & config

None at this level. Eleven of the fifteen samples need no permission at all;
the two location samples and the data monitor are where the permission work
lives, and each card states its own.

## Constraints

- The catalog carries **no ordering and no dependency information** - the
  fifteen entries are peers, and nothing here says which to read first.
- There is **no framework project** for this industry, so nothing about module
  layering, HAR boundaries or shared utilities can be taken from these
  documents. Each sample is a single `entry` module.
- The links point at the live developer site, so a scenario added there after
  this snapshot would not appear in the corpus.

## Pitfalls

None in the document itself - the catalog is accurate, complete and matches
both the downloaded samples and the sidebar tree.

## References

- `huawei_industry_tree/08_children_education/docs/01_practice-kids-app-architecture-v1_1.md` - the page itself
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-kids-app-architecture-v1_1-0000002270219009
- `huawei_industry_tree/08_children_education/tree.json` - the sidebar, which matches the catalog
- `KIDS-17` - the industry FAQ page, which holds no content
- `SPORT-02`, `TOUR-02` - the equivalent catalog pages, both of which sit under an architecture document this industry does not have
