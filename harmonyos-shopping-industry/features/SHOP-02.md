---
id: SHOP-02
title: Key scenario index - the 21 shopping samples that fill in the architecture skeleton
industry: 16_shopping
doc: huawei_industry_tree/16_shopping/docs/02_practice-purchase-app-architecture-v1_1.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-purchase-app-architecture-v1_1-0000002270637853
sample: none
kits: []
apis: []
permissions: []
min_api: 20
modules: []
findings: []
status: verified
---

## When to use

Load this card when you need to **find the right shopping sample**, not to
build anything. The page it documents is a bare link list: twenty-one key
scenarios (关键场景示例) hanging off the shopping architecture guide, each one
a separate document with its own downloadable zip.

`SHOP-01` gives you the skeleton - the module layering, the responsive shell,
the scan and image-search entries - and stops there, because the document
itself says it publishes only framework code. This index is the other half:
each entry is a self-contained page-level feature you drop into that skeleton.
Read it as the industry's feature catalogue, and use the mapping below to jump
straight to the card for the scenario you need.

**Verification level is low for this card.** There is no sample, no code and
nothing to compile. What can be checked is that the twenty-one links resolve to
the twenty-one scenario documents we hold, and they do - the list is complete
and its order matches the docs directory exactly.

## Feature checklist

What the page promises, and what we could confirm:

- Twenty-one links, one per key shopping scenario. **Confirmed** - the list has
  exactly 21 entries.
- Each link points at an `architecture-guides` document under
  `developer.huawei.com`. **Confirmed.**
- Every linked scenario exists in our mirror as `docs/03..23`. **Confirmed** -
  one-to-one, in the same order, no gaps and no extras. Note that the industry
  also holds a 24th document (`SHOP-24`, the v1_2 architecture page) that this
  index does not list, because it is not a scenario.
- Each scenario document carries its own code-download link. **Confirmed** for
  all 21; the zips are in `huawei_industry_tree/16_shopping/downloads/`.
- The page carries no scenario descriptions, no grouping, and no indication of
  which scenarios depend on each other. **Confirmed** - that is the page's main
  weakness.

## Architecture

No project, no modules. The document is a flat unordered list of 21 anchors and
nothing else - no prose, no constraints section, no reference section.

The mapping from the page's list order to this skill pack:

| # | Scenario | Card | Sample |
| --- | --- | --- | --- |
| 1 | 优惠券卡包 - coupon wallet | `SHOP-03` | CouponsPage.zip |
| 2 | 签到、完成浏览任务获取积分 - check-in and points | `SHOP-04` | Sign.zip |
| 3 | 优惠券防重复领取 - one coupon per user | `SHOP-05` | GetCoupons.zip |
| 4 | 基于Refresh实现下拉跳转到指定页面 - pull-down to jump | `SHOP-06` | PullToJump.zip |
| 5 | 刮刮乐抽奖效果 - scratch card | `SHOP-07` | scratch.zip |
| 6 | 商品长按标记 - long-press product marking | `SHOP-08` | 商品长按标记示例代码.zip |
| 7 | 搜索及编辑搜索历史 - search history editing | `SHOP-09` | SearchHistory.zip |
| 8 | 商品页面刷新和展示 - sticky refresh feed | `SHOP-10` | ScrollCellingDemo.zip |
| 9 | 优惠券页面骨架屏效果 - skeleton screen | `SHOP-11` | SkeletonScreen.zip |
| 10 | 搜索历史展开与收起 - expanding history | `SHOP-12` | SearchHistoryFolding.zip |
| 11 | 订单状态切换与收货后评价 - order states and reviews | `SHOP-13` | EvaluationAfterReceived.zip |
| 12 | 首页菜单横向滚动高度自适应 - self-sizing menu | `SHOP-14` | AutoHeightList.zip |
| 13 | 商品浏览记录存储 - browsing history storage | `SHOP-15` | OfferingBrowsingHistory.zip |
| 14 | 抢票倒计时及提醒 - ticket-grab countdown | `SHOP-16` | CountDownDemo.zip |
| 15 | 红包雨 - red envelope rain | `SHOP-17` | RedEnvelopeRain.zip |
| 16 | 未读消息清除 - clearing unread badges | `SHOP-18` | ClearUnreadMessages.zip |
| 17 | 销售数据实时刷新 - live sales figures | `SHOP-19` | SalesUpdateRoll.zip |
| 18 | 搜索推荐词轮播 - rotating search hints | `SHOP-20` | RecommendSearch.zip |
| 19 | 商品对比查询 - product comparison | `SHOP-21` | GoodsPK.zip |
| 20 | 首页下拉进入二楼 - pull-down "second floor" | `SHOP-22` | SecondFloor.zip |
| 21 | 地址列表与地址管理 - address book | `SHOP-23` | AddressManager.zip |

**The design decision worth noticing** is that these are *page-level* scenarios,
not components. Each zip is a whole `entry` module with its own `EntryAbility`
and one page, which is why they can be read in isolation and why almost none of
them share code. The cost is duplication: the same avoid-area boilerplate, the
same `StyleConstants` class and the same rawfile-JSON loading pattern are
re-implemented in nearly every sample. When you adopt more than two of them,
lift that boilerplate into a `common` HAR the way `SHOP-01` does, before the
copies diverge.

The list also has a shape worth reading. Four scenarios are about **promotions**
(3, 5, 7, 17), five about **search and history** (7, 9, 10, 18, 20), four about
**feed and list mechanics** (8, 10, 12, 14), three about **orders and delivery**
(11, 13, 21) and the rest are one-off interaction effects. If you are building a
shopping app from scratch, the search-and-history cluster is the one to read
together - those five samples solve overlapping problems in incompatible ways.

## Implementation steps

There is nothing to implement from this page. The useful procedure is how to
consume it:

1. **Start from `SHOP-01`** for the module layout, the breakpoint mechanism and
   the navigation model. Those decisions are hard to retrofit.
2. **Pick scenarios by the table above,** not by browsing the zips - the sample
   directory names do not always match the scenario titles
   (`SearchHistoryFolding.zip` for 搜索历史展开与收起, for instance).
3. **Read the scenario card before the scenario document.** Every one of these
   docs abridges its code snippets, and a large share of the abridged snippets
   no longer parse (`HW-16-0013`, catalogued on `SHOP-12`); the cards quote the
   zips instead.
4. **Check each sample's project tree against its zip** before following it -
   the tree in the document is stale in several of these scenarios
   (`SHOP-03`, `SHOP-04`).
5. **Merge the boilerplate once.** The avoid-area publishing, the
   `StyleConstants` classes and the rawfile loaders are duplicated across
   nearly every zip; consolidate them at adoption time.

## Verified snippets

**No code ships with this page.** The document contains no snippets at all -
only anchors. The entire body, reproduced from the doc, is twenty-one lines of
the form:

```markdown
- **[优惠券卡包](https://developer.huawei.com/consumer/cn/doc/architecture-guides/coupons_page-0000002235638646)**
- **[签到、完成浏览任务获取积分](https://developer.huawei.com/consumer/cn/doc/architecture-guides/sign-0000002284183885)**
- **[优惠券防重复领取](https://developer.huawei.com/consumer/cn/doc/architecture-guides/get_coupons-0000002256272966)**
```

(from the doc — no sample shipped; not compile-verified, and there is nothing
to compile.)

For code, go to the card named in the table. Every one of them quotes its
snippets from the extracted zip rather than from the document.

## Pitfalls

- **No defects found in this document.** All 21 links resolve to scenario
  documents we hold, the list is complete, and there is no code to be wrong.
- The page carries **no descriptions**: the reader gets 21 Chinese titles and
  must open each document to learn what the scenario does. The table above
  exists to spare that.
- It also carries **no constraints section**, unlike every scenario document it
  links to. The shared floor is API Version 20 Release / HarmonyOS 6.0.0
  Release SDK / DevEco Studio 6.0.0 Release, stated individually in each child
  document.
- Being an index, this page ages badly: it is the file to re-crawl first when
  checking whether the industry has gained scenarios.

## References

- `huawei_industry_tree/16_shopping/docs/02_practice-purchase-app-architecture-v1_1.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-purchase-app-architecture-v1_1-0000002270637853
- `SHOP-01` - the architecture skeleton these scenarios plug into
- `SHOP-03` .. `SHOP-23` - one card per linked scenario, per the table above
- `huawei_industry_tree/16_shopping/downloads/` - the 21 sample zips
