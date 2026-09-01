# HarmonyOS Industry Skills

Claude Code skills for building HarmonyOS applications, one per industry.

Each one carries Huawei HQ's feature catalogue for that vertical, verified code
taken from the compiled sample projects, and the defects we found in HQ's own
published documentation. There are **1764 evidence-backed findings** behind
these folders, from a review of 443 documents and 399 sample projects across 19
industries.

## Install

Claude Code discovers skills under `.claude/skills/`. Installation is one drag:

```
YourApp/
└── .claude/
    └── skills/
        └── harmonyos-education-industry/
```

Copy the folder for your industry. Restart Claude Code, then run `/skills` to
confirm it is listed.

That is the whole installation. Each industry skill is self-contained: no
scripts, no sample archives, no shared folder. Every reference carries the
canonical Huawei URL beside it. Sizes run from 250 KB to about 2 MB.

`harmonyos-hq-base` is the exception - do not install it. Its content is
already copied inside every industry skill as `references/base/`. It is kept
here as the source those copies are built from.

## Use an industry skill

Ask for what you are building. Claude loads the skill on its own, because the
`description` in its `SKILL.md` says when it applies.

```
Build the weekly timetable screen.
```

What Claude then does, and what you should check it did:

1. Opens `references/feature-catalog.md` and names the feature IDs it matched.
2. Reads `features/<ID>.md` in full before writing code.
3. Copies from the card's **Verified snippets** - real code from the compiled
   sample project, not from the published document, which is abridged and in
   places wrong.
4. Checks the card's **Pitfalls** section before finishing.

You can also name the feature directly: `Implement EDU-04.`

## Generate documentation

`harmonyos-doc-generator` writes a document set for your application in the
same format Huawei publishes for its own industry guides: an application case,
one document per key scenario, a scenario index, and a runnable minimal
project for each scenario.

Install it **next to** the industry skill - it reads the industry skill's
cards, so it does not work alone:

```
.claude/skills/
├── harmonyos-doc-generator/
└── harmonyos-education-industry/
```

Then type:

```
generate docs
```

It asks four things - industry, application name, target API level, and
whether your application's source is available - and then stops to show you
the scenarios it selected before writing anything.

### It works without your source code

Scenarios come from the industry skill, not from your application. The cards
already hold verified code, so a full document set can be produced on a
machine where the application's source cannot be reached at all.

- **No source available.** Everything comes from the cards. The application
  case describes the industry's reference architecture rather than a specific
  product, and says so.
- **Source available.** Same selection rules; where your application
  implements a scenario, its own code is used instead of the card's.

Alignment, not dependence: a scenario stays in the set even when your
application has not built it, and does not enter the set only because it has.

### How many scenarios you get

Between 10 and 18, targeting 12 to 15. Every candidate passes four gates -
it runs standalone, it can be stated in one sentence without naming a screen,
it has verified code, and it is not a navigation page. Survivors are then
ranked, most importantly on whether implementing it correctly needs knowledge
that is not already on the component's reference page.

At most three scenarios per capability cluster, so a map-heavy industry does
not produce six map scenarios and nothing else.

Before writing, it shows you what it selected and what it rejected, with the
reason for each. That table is where you steer the set.

## What is inside a skill

```
harmonyos-<industry>-industry/
├── SKILL.md                     when to load it, the shared rules, how to use it
├── features/<PREFIX>-NN.md      one card per feature
├── references/
│   ├── feature-catalog.md       every feature in one table - start here
│   ├── api-map.md               feature to kit / permission / API level
│   ├── pitfalls.md              the findings, systemic ones first
│   └── base/                    rules shared across all industries
│       ├── layered-architecture.md
│       ├── module-types.md      HAP vs HAR vs HSP
│       ├── navigation.md        one UIAbility, Navigation over Router
│       ├── state-management.md
│       ├── performance.md
│       ├── permissions.md
│       └── project-skeleton/    a three-tier project in its smallest form
└── EVIDENCE.md                  each feature's source document and sample project
```

A card holds: when to use the feature, a behaviour checklist, the module
layout, implementation steps, verified code, permissions, constraints, the
defects found in HQ's document, and links to the official pages.

## Industries

| Industry | Skill | Feature prefix | Features |
|---|---|---|---:|
| Automotive | `harmonyos-auto-industry` | `AUTO-` | 8 |
| Convenient life | `harmonyos-convenient-life-industry` | `LIFE-` | 31 |
| Sports and health | `harmonyos-sports-health-industry` | `SPORT-` | 15 |
| Education | `harmonyos-education-industry` | `EDU-` | 21 |
| Office | `harmonyos-office-industry` | `OFFICE-` | 32 |
| Public transport | `harmonyos-public-transport-industry` | `TRANS-` | 9 |
| Finance and insurance | `harmonyos-finance-insurance-industry` | `FIN-` | 11 |
| Children's education | `harmonyos-children-education-industry` | `KIDS-` | 17 |
| Tourism | `harmonyos-tourism-industry` | `TOUR-` | 13 |
| Maternity and health | `harmonyos-maternity-health-industry` | `MAT-` | 5 |
| News and reading | `harmonyos-news-reading-industry` | `NEWS-` | 27 |
| Jobs | `harmonyos-jobs-industry` | `JOBS-` | 5 |
| Media and entertainment | `harmonyos-media-entertainment-industry` | `MEDIA-` | 43 |
| Social communication | `harmonyos-social-communication-industry` | `SOCIAL-` | 45 |
| Utilities | `harmonyos-utilities-industry` | `UTIL-` | 47 |
| Shopping | `harmonyos-shopping-industry` | `SHOP-` | 24 |
| Food | `harmonyos-food-industry` | `FOOD-` | 6 |
| Photography | `harmonyos-photography-industry` | `PHOTO-` | 32 |
| Common technical solutions | `harmonyos-common-industry` | `COMMON-` | 52 |

Plus `harmonyos-doc-generator` (documentation) and `harmonyos-hq-base` (the
source of `references/base/`, not installed).

## What these do not do

- They do not write the application. This is a reference, not a generator.
- Payment, authentication, push and backend are out of scope. The content
  follows Huawei's industry documents; what is not there is not here.
- No visual design, no test strategy.
- They age. If Huawei corrects a document, the card becomes stale - every card
  carries the canonical URL so it can be compared.
- The value depends on the card being read. Skipping the card and copying from
  a web search gets you nothing.

## Where this came from

Huawei publishes industry guides and downloadable sample projects, presented
as templates and copied wholesale into real products. We read every document
against its sample project and against the official HarmonyOS guides.

A finding counts as confirmed only with a verbatim quote from official
documentation or a real line from a sample project.

| Severity | Findings | Meaning |
|---|---:|---|
| blocker | 23 | Following the document as written crashes, fails to build, or loses data |
| high | 319 | Wrong behaviour, a leak, or a security weakness a developer would copy |
| medium | 909 | Misleading guidance; it runs, but contradicts HQ's own documentation |
| low | 513 | Cosmetic, inconsistent terminology, dead link |

Generated content - `references/`, `EVIDENCE.md`, `SKILL.md` - is built from
the cards in the review repository. Do not edit it here by hand.
