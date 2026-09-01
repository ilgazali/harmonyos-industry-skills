# Choosing key scenarios

Two things decide the quality of the document set: which scenarios you pick,
and how many. Both are rule-based here. Do not improvise either.

## Where scenarios come from

The industry skill is the primary source, not the application. Its
`references/feature-catalog.md` lists the capabilities Huawei treats as key
scenarios for this vertical, and each `features/<ID>.md` card carries verified
code taken from a compiled sample project, with the published document's
defects already corrected.

This is what lets the set be produced without access to the application's
source at all.

### Mode 1 - no application source

Everything comes from the industry skill. Scenarios are chosen from the
catalog; code blocks come from each card's **Verified snippets**; the
application case describes the layered architecture the industry uses rather
than a specific product.

Say so in the application case: the document set describes the industry's
reference implementation, not a shipped product.

### Mode 2 - application source available

Same catalog, same selection rules, one addition: a scenario the application
actually implements outranks one it does not, and its code blocks come from
the application rather than from the card.

Alignment, not dependence. A scenario worth documenting stays in the set even
when the application has not built it - the card's verified code carries it.
Never drop a scenario only because the application lacks it, and never add
one only because the application has it.

## How many

**Between 10 and 18. Target 12 to 15.**

The application case is separate and is not counted. It is `main-industry-doc.md`
and takes no number, so the scenarios run from `01` and the highest scenario
number equals the count.

Thirty scenarios is not a thorough document set, it is an unfiltered one. If
more than 18 candidates survive the gates, the ranking below decides which
ones stay. If fewer than 8 survive, write fewer and say why - do not pad the
set to reach a number.

## Gates

A candidate must pass all four. No score, no exceptions.

**1. Standalone.** It runs in one empty entry module. If it needs session
state, the application's data model, or another screen, it fails.

**2. Capability sentence.** One sentence, no screen name.

- Passes: "Converts a PDF into one long image and saves it to the gallery."
- Fails: "Opens the menu in the top right corner of the library page."

**3. Verified code exists.** Either the card's Verified snippets or the
application's source. A scenario you would have to write fresh code for does
not qualify - that code has never been compiled or run, and documenting it
would repeat the mistake this whole corpus exists to correct.

**4. Not navigational.** Scenario indexes and industry FAQs are not
scenarios. The industry's architecture card (`XXX-01`) is not one either - it
becomes `main-industry-doc.md`, the application case.

## Ranking

Score the survivors on these four, in this order of weight.

**1. Teaching value.** Does implementing it correctly need knowledge that is
not on the component's reference page?

A card carrying findings scores highest: the published version got it wrong,
and what the card corrects is exactly the knowledge worth writing down. A
scenario that is one API call with default options scores zero.

**2. Reuse breadth.** Would another application in this industry want it? A
capability that only makes sense inside one product ranks low.

**3. Composition.** Does it combine at least two components or APIs, or
manage a lifecycle - subscribe and unsubscribe, a permission flow, resource
release? Single-component demonstrations rank low.

**4. Application alignment.** Mode 2 only. Among otherwise equal candidates,
prefer the one the application implements.

## Spread

At most **three scenarios per capability cluster**. Cluster by primary kit,
using the industry skill's `references/api-map.md`.

Without this rule a map-heavy industry produces six map scenarios and nothing
else, and the set stops representing the vertical.

## Never document

- A single component with default properties
- One API call with no lifecycle, error path or permission handling
- Anything the official reference page already covers completely
- A screen that is layout only, with no behaviour
- Login, settings, profile and other ordinary application furniture

## Naming

Follow the phrasing in [scenario-index.md](scenario-index.md): name the
capability, not your product.

- Good: `Course reminders in the system calendar`
- Bad: `MyCourses page reminder button`

Internal names - class, module, screen - never appear in a title. They appear
in code blocks, because that is where you are showing real code.

## Sizing

A scenario document's implementation approach runs 3 to 6 numbered steps.
More than that means the scenario is too large; split it. Fewer than three
usually means it is a detail of a larger one; merge it.

## Ordering the index

Related scenarios sit next to each other. A workable default: basic layout
and navigation first, then data and persistence, then media, then device
capabilities. If the scenarios form a chronological flow - pick dates, set
the route, confirm details, look the order up later - keep that order
instead; it reads better than any categorical grouping.

## Report the selection

Before writing any document, show the user a table:

| ID | Scenario | Cluster | Why it is in |
|---|---|---|---|

And below it, the candidates you rejected with the gate or ranking reason.
The rejections matter as much as the selection - they are what keeps the set
at 15 instead of 30.
